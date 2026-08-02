# TSGCTF2021 lkgit WP

## 题目简述

`lkgit` 是一个内核模块，使用 16 字节弱哈希保存最多 0x30 个对象：

```c
typedef struct {
    char hash[0x10];
    char *content;
    char *message;
} hash_object;
```

哈希只是把 64 字节内容按每 4 字节一组异或：

```c
for (ix = 0; ix != 0x10; ++ix) {
    c = 0;
    for (jx = 0; jx != 4; ++jx)
        c ^= content[ix * 4 + jx];
    hash[ix] = c;
}
```

发生哈希冲突时，模块立即 `kfree` 旧 `hash_object`，但正在执行的 `GET_OBJECT` 或 `AMEND_MESSAGE` 仍可能持有该指针。官方 EXP 用 userfaultfd 把内核线程停在 `copy_to_user/copy_from_user` 中，在另一线程制造冲突，形成可控 UAF。

## 解题过程

### 1. 用 userfaultfd 扩大竞争窗口

在用户空间映射两页，把第二页注册为 missing page，再让 `log_object` 的 `message` 字段恰好落入该页。调用 ioctl 时，内核先找到目标并保存：

```c
target = objects[target_ix];
```

当稍后的复制操作访问 missing page，处理该 ioctl 的内核线程会睡眠；userfaultfd 处理线程此时可以发起另一条 ioctl。弱 XOR 哈希让构造相同哈希的不同内容非常容易。

### 2. 第一次 UAF 泄露内核基址

先保存一个正常对象，再调用 `LKGIT_GET_OBJECT`。让执行停在向用户复制 `message` 的阶段，然后提交冲突对象：

```c
if ((dup_ix = find_by_hash(obj->hash)) != -1) {
    kfree(objects[dup_ix]);
    objects[dup_ix] = NULL;
}
```

旧 `target` 成为悬空指针。处理线程随后大量打开 `/proc/self/stat`，使同为 kmalloc-32 的 `seq_operations` 结构复用该块。解除缺页后，原 `lkgit_get_object` 继续把 `target->hash` 等字段复制给用户，实际泄露的是 `seq_operations` 中的函数指针 `single_start`：

```c
single_start = *(unsigned long *)log->hash;
kernbase = single_start - 0x1adc20;
```

由此得到 KASLR 后的内核基址，并定位：

```text
modprobe_path = kernbase + 0x0c3cb20
```

### 3. 第二次 UAF 写 `modprobe_path`

第二轮把 `LKGIT_AMEND_MESSAGE` 停在：

```c
copy_from_user(buf, reqptr->message, 0x20);
```

函数在睡眠前已保存旧 `target`。userfaultfd 线程再次提交冲突对象释放它，然后安排一个 kmalloc-32 的消息缓冲区复用同一槽位；该缓冲区的每个 8 字节都填入 `modprobe_path` 地址。旧内存若仍被解释成 `hash_object`，其 `message` 指针就等于 `modprobe_path`。

处理缺页时向用户页填入 `/tmp/evil\0`。原 ioctl 恢复后，栈上 `buf` 得到该字符串，最后执行：

```c
memcpy(target->message, buf, MESSAGE_MAXSZ);
```

由于 `target` 已被替换，这行代码实际把 `/tmp/evil` 写入内核全局 `modprobe_path`。

### 4. 触发 modprobe 提权

EXP 预先创建：

```sh
/tmp/evil      # root 脚本，放宽 /home/user/flag 权限并输出 flag
/tmp/nirugiri  # 以 ff ff ff ff 开头的未知格式可执行文件
```

执行 `/tmp/nirugiri` 时，内核按新的 `modprobe_path` 以 root 身份启动 `/tmp/evil`。最终读取：

```text
TSGCTF{google_took_2years_but_you_found_hash_collision_in_a_day!}
```

## 方法总结

本题的根因不是弱哈希本身，而是冲突替换在没有引用计数或锁定生命周期的情况下释放对象。userfaultfd 把很窄的竞态窗口稳定扩展为可编排事件：第一次让悬空对象复用为 `seq_operations` 以泄露 KASLR，第二次让它复用为攻击者控制的 kmalloc-32 数据并形成任意地址写，最后修改 `modprobe_path` 提权。修复应为对象访问加锁或引用计数，并在替换前确保没有在途读写；哈希也必须具备抗碰撞性，但这不能代替正确的并发生命周期管理。
