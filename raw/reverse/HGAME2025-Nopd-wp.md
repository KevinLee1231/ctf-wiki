# Nop'd

## 题目简述

附件包含两个 Linux x86-64 ELF：`game` 是一个数值平衡很差的文本地牢游戏，`launcher` 看似只是带参数启动器。单独运行 `game` 时只能看到普通游戏流程；通过 `launcher` 启动时，同一份 `game` 会出现额外行为。

真正机制类似半主机（semihosting）：`game` 用特殊多字节 NOP 包围 `syscall`，把它们编码成“宿主调用”；`launcher` 通过 `ptrace` 监视子进程、解码这些 NOP、从寄存器或栈中取参，并在宿主进程里执行隐藏函数。最终被拆散的操作组成 ChaCha20 和一层字节链式异或。

[作者公开源码仓库](https://github.com/CSharperMantle/hgame2025_nopd_public)保留了 `guest`、`host` 和宏定义，可用于核对从二进制恢复出的协议；以下正文已完整说明协议和解密参数。

## 解题过程

`launcher` 会先 `fork`，子进程 `execv` 目标程序，父进程把 PID 存入全局变量。初始化函数把一个未被 IDA 自动识别的函数注册成 `atexit` handler；当父进程的 `main` 返回后，handler 根据 `pid > 0` 进入真正的监视循环。

监视器使用 [`ptrace`](https://man7.org/linux/man-pages/man2/ptrace.2.html) 的 `PTRACE_SYSCALL` 在系统调用进入、退出时暂停子进程，并用 `PTRACE_GETREGS` 读取 `struct user_regs_struct`。每次暂停时，它检查 `syscall` 前面的 6 字节是否满足：

```text
[0x40-0x43] 0x0f 0x1f 0x44 <SIB> 0x7f
```

这是 [x86-64 多字节 NOP](https://www.felixcloutier.com/x86/nop) 的一种编码，可写成：

```asm
nop dword ptr [base + index * scale + disp8]
```

一条完整的隐藏调用具有以下结构：

```asm
nop dword ptr [rax + rax*1 + 0x7f]  ; 开始标记
syscall
nop dword ptr [base + rax*scale + argument_index]
...
nop dword ptr [rax + rax*1 + 0x7e]  ; 结束标记
```

`syscall` 后的 NOP 描述参数来源和参数序号：

- SIB 的 scale 编码为 `0`，即实际倍率为 `1` 时，从 base 指定的子进程寄存器取值。
- scale 编码非零时，从 `rsp + 8 * base` 指定的栈槽取值。
- `disp8` 表示参数编号；第一个有效参数是宿主处理函数编号。

启动器把参数值写入 `qword_5120`，把有效位写入 `byte_5540`，再根据 `qword_5120[0]` 从函数表选择处理器。恢复后的函数表为：

| 编号 | 处理器作用 |
| ---: | --- |
| 0 | 空操作 |
| 1 | 从子进程输入，即 `gets` |
| 2 | 向子进程输出，即 `puts` |
| 3 | ChaCha quarter round，SIMD 并行处理四个 lane |
| 4 | 重排 ChaCha 状态矩阵，在列轮和对角轮之间切换 |
| 5 | 初始化 ChaCha20 状态 |
| 6 | 将轮函数结果与初始状态逐字相加 |
| 7 | 比较输入与目标密文，即 `memcmp` |
| 8 | 返回常数 `0x61C88646`，后续只使用低字节 `0x46` |
| 9 | 修改子进程 RIP，跳过后续 128 字节 |

处理完成后，`launcher` 把子进程的 `orig_rax` 改为 `-1`，让内核执行一个无效系统调用，从而抑制原始 syscall 的效果；到 syscall-exit 再把宿主函数结果写回 `rax`，并恢复或修改后的 `rip`。因此离开 `launcher` 后，多字节 NOP 仍然是无害 NOP，`game` 也仍能作为普通程序运行。

在 IDA 中可用 “Decompile as call” 恢复这些调用。例如，参数来自 `rbx`、`r8` 时：

```c
uint64_t __usercall nopcall_r8@<rax>(
    uintptr_t@<rbx>,
    uintptr_t@<r8>
);
```

若还包含相对栈顶 `8` 和 `0` 字节处的参数：

```c
uint64_t __usercall nopcall_r9s8s0@<rax>(
    uintptr_t@<rbx>,
    uintptr_t@<r9>,
    uintptr_t@<^8>,
    uintptr_t@<^0>
);
```

继续恢复宿主函数，可以从加法、异或以及循环左移 `16/12/8/7` 位识别出 ChaCha quarter round。矩阵重排函数在列轮与对角轮间调整 lane；初始化函数构造 RFC 7539 ChaCha20 状态。初始化常量在二进制中每字节加了 `4`，还原后是标准字符串：

```text
expand 32-byte k
```

动态跟踪宿主处理器的调用顺序，可以观察到一次初始化，随后 20 次 quarter round 与 20 次重排，最后执行块加法。这与 ChaCha20 的 20 轮，即 10 个 double round 完全一致。

ChaCha20 的另外三项参数也能直接从 `game` 中恢复：

```text
key     = "It's all written in the Book of "  # 32 字节
nonce   = "What's your "                      # 12 字节
counter = 0
```

其中 key 和 nonce 分别来自两个正常提示字符串的前 32、12 字节。宿主编号 `9` 会跳过一条 `hlt` 和填充区，使控制流到达最终校验。编号 `8` 给出初始链值 `0x46`。随后程序逐字节执行：

```c
last = 0x46;
for (size_t i = 0; i < 64; i++) {
    input[i] ^= keystream[i];
    input[i] ^= last;
    last = input[i];
}
```

设明文字节为 $P_i$、ChaCha20 密钥流为 $K_i$、输出为 $C_i$，且 $C_{-1}=0x46$，则：

$$
C_i=P_i\oplus K_i\oplus C_{i-1}.
$$

因此逆变换为：

$$
P_i=C_i\oplus C_{i-1}\oplus K_i.
$$

程序用于比较的 51 字节目标数据为：

```text
64 6a 50 17 81 7d 6f 1a 87 b1 a4 00 09 03 f8 8d
f8 6b df 32 5f 40 90 9c b8 3d 86 13 26 b7 63 f7
74 e8 53 ed 58 20 4f d9 99 26 21 37 de 35 76 c8
bc d0 6e
```

注意中间是 `f8 8d f8`；PDF 中超长 CyberChef URL 的换行文本容易漏看一个半字节，应以二进制中的 `RAND_POOL` 或作者源码为准。完整解密脚本如下：

```python
from Crypto.Cipher import ChaCha20

key = b"It's all written in the Book of "
nonce = b"What's your "

ciphertext = bytes.fromhex(
    "64 6a 50 17 81 7d 6f 1a 87 b1 a4 00 09 03 f8 8d "
    "f8 6b df 32 5f 40 90 9c b8 3d 86 13 26 b7 63 f7 "
    "74 e8 53 ed 58 20 4f d9 99 26 21 37 de 35 76 c8 "
    "bc d0 6e"
)

# PyCryptodome 对 12 字节 nonce 的 ChaCha20 从 counter=0 开始。
stream = ChaCha20.new(key=key, nonce=nonce).encrypt(bytes(len(ciphertext)))

previous = 0x46
plaintext = bytearray()

for current, key_byte in zip(ciphertext, stream):
    plaintext.append(current ^ previous ^ key_byte)
    previous = current

print(plaintext.decode())
```

输出 flag：

```text
hgame{D3n1ably-c0mmunicate-by-d0ing-m@g1cal-no-op!}
```

## 方法总结

本题的障眼法不是单纯的 NOP 填充，而是一套由 `launcher` 实现的半主机协议。`game` 中的特殊 NOP 编码参数来源和顺序，`syscall` 只是触发 `ptrace` 监视器的陷阱；脱离监视器时这些指令仍保持无害，形成可抵赖的正常程序。恢复函数表后，20 轮 quarter round、矩阵重排、标准常量和块加法共同指向 ChaCha20。最后再逆掉以 `0x46` 为初值的字节链式异或即可得到 flag。动态跟踪调用顺序最快，静态分析则必须同时理解 x86-64 NOP/SIB 编码、`ptrace` 寄存器语义和自定义调用约定。
