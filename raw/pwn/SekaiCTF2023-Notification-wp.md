# Notification

## 题目简述

程序接收一串带超时、长度、子命令偏移和 ID 的任务，支持提交任务、按 ID 取消任务，以及查看处理完成后的消息。它不用 glibc `malloc` 管理主要对象，而采用 [libzone](https://github.com/peternguyen93/libzone) 的类型隔离 zone allocator。

由官方 solver 的封包函数可还原 16 字节任务头：

```python
header = flat(
    p16(timeout),
    p16(buffer_length),
    p16(child_offset),
    p16(0xff),
    p64(task_id),
)
```

父任务会通过 `child_offset` 指向同一输入中的下一条命令。主漏洞是：创建子命令时没有检查 ID 是否已存在。两个同 ID 任务会使队列删除错误对象，留下已释放的任务指针。另有一处子命令长度检查错误，使信息泄露阶段更容易完成。

## 解题过程

### 1. 理清 libzone 的页面策略

外部仓库的 `libzone.c` 表明分配器有两套接口：

- `zone_create`、`zone_alloc`、`zone_free` 管理显式命名、固定对象类型的 zone；
- `zmalloc`、`zfree` 按申请大小选择或创建 `zmalloc.SIZE` zone。

每个 zone 由若干独立 `mmap` 页面组成。没有可用槽位时映射新页；存在已释放槽位时，分配器优先选择 `mapped_num_free` 最小的页面，即最接近填满的页面。当某页中曾分配的槽位几乎全部释放，且该 zone 已有至少 3 页时，`zone_free_internal` 会把整页 `munmap`。

这些行为意味着类型隔离并非绝对安全：只要让含悬空指针的页面被解除映射，下一次其他 zone 的 `mmap` 很可能复用同一虚拟地址，旧指针就会把新类型对象解释成旧类型对象。

### 2. 用重复 ID 制造任务 UAF

先喷射约 3 页 `task` 对象。官方脚本保留一个短超时旧任务 `0x47d`，随后通过父子命令再创建一个 ID 同为 `0x47d` 的新任务：

```python
submit_1(2, 0, 0x47d, b"")  # 较早的短超时任务

submit_2(
    0xff, 0, 0x7c, b"",
    0,    0, 0x47d, b"",     # 子命令未检查 ID 重复
)
```

旧任务超时后，处理线程释放旧对象；但它按 ID 遍历队列删除记录时，先删掉了新对象的队列节点。于是已经释放的旧任务仍留在队列中，形成悬空指针。

接着释放该 task zone 第三页上的其余对象，使页面满足 libzone 的整页回收条件。旧指针仍保存原虚拟地址，但该地址现在已经没有映射。

### 3. 连续两次跨 zone 类型混淆取得泄露

第一轮复用把 `out_message` zone 的新页映射到刚刚释放的 task 页地址。队列中的悬空 `task*` 实际指向一个 `out_message`。此时按旧任务 ID 执行取消，程序按 task 布局释放所谓的 `inline_mes`；对应字段其实是 `out_message` 中的缓冲区指针，于是又制造出一个悬空消息缓冲区。

第二轮对 0x80 大小的普通缓冲区重复“三页填充 → 清空最后一页 → `munmap`”，再喷射 task 对象占据同一地址。`check_message` 仍把悬空缓冲区当作消息正文打印，实际输出的却是 task 结构内容。官方脚本跳过前 `0x18` 字节后读取一个指针：

```python
check()
io.recvuntil(b"ID 0x137: ")
io.recvn(0x18)
mapping_reference = u64(io.recvn(8)) - 0x90
```

这个环境相关的映射基准随后用于计算目标函数地址；官方构建中 `system` 对应：

```python
system_address = mapping_reference + 0x86ad60
```

这里的差值绑定于题目 Ubuntu 22.04 容器及其二进制、共享库布局，不能当成通用 libc 偏移。

### 4. 伪造任务并劫持取消回调

再次利用重复 ID 留下悬空 task 指针，并把所在 task 页完全解除映射。随后提交一个此前未使用的 0x40 长度，让 `zmalloc(0x40)` 为新的大小类映射页面，正好覆盖旧 task 地址。

输入缓冲区按 task 布局伪造：

```python
fake_task = (
    p64(0x6873)              # 小端字节为 b"sh\x00..."
    + p64(0x40)
    + p64(0) * 3
    + p64(system_address)    # cancel_callback
).ljust(0x40, b"\x00")

submit_cmd(fake_task)
cancel(0x6873)
```

数值 `0x6873` 同时满足两个用途：程序把它当作任务 ID 查找；在内存中其低字节又组成字符串 `sh\x00`。取消任务时，程序以对象指针为参数调用受控的 `cancel_callback`，实际执行 `system(fake_task)`，也就是 `system("sh")`。

取得 shell 后读取：

```text
SEKAI{type_confusion_in_object_segregation_heap_f1d5f1197eb97c6496aff578f4c4083963d26246}
```

## 方法总结

对象按类型分 zone 只能阻止同一映射内的直接异型复用，不能修复应用层 UAF。Notification 的核心是精确操纵 libzone 的页计数、空闲槽选择和整页 `munmap`，让 Linux 把同一地址重新映射给另一种 zone：`task → out_message → 普通缓冲区 → task`。重复 ID 提供稳定悬空指针，消息打印提供泄露，最后把回调字段改为 `system` 并让对象开头同时充当 `"sh"` 参数。
