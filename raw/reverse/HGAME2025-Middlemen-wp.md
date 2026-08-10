# Middlemen

## 题目简述

这是一个 Android ARM64 逆向题。Java 层把用户输入交给动态注册的 native 方法 `middlemen`，native 层把 flag 主体解析为 16 个十六进制字节，然后调用 AArch64 的 `getpid` 系统调用（编号 `172`）进行校验。

表面上 `getpid` 与 flag 无关，真正的检查被藏在 seccomp-BPF 过滤器和 `SIGSYS` 信号处理函数中：BPF 先约束输入的后 8 字节，满足条件时返回 `SECCOMP_RET_TRAP`；信号处理函数再从 `ucontext_t` 取回输入，派生 AES 密钥并校验全部 16 字节。

## 解题过程

`middlemen` 先按 UUID 形式读取 16 个字节，随后调用：

```c
syscall(
    172,
    input_qword_0,
    *(uint64_t *)(input + 4),
    input_qword_1,
    *(uint64_t *)(input + 12),
    0x221221
);
```

在 `.init_array` 引用的初始化函数中可以看到：

```c
prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog);
sigaction(SIGSYS, &action, NULL);
```

其中 `prctl(22, 2, &prog)` 对应 `PR_SET_SECCOMP + SECCOMP_MODE_FILTER`，而信号号 `31` 是 `SIGSYS`。将 BPF 字节码导出为 `filter.bpf` 后，可以使用 [seccomp-tools](https://github.com/david942j/seccomp-tools) 反汇编：

```bash
seccomp-tools disasm filter.bpf
```

过滤器的有效逻辑可整理为：

```c
if (arch != ARCH_AARCH64)
    return ALLOW;

uint32_t v0 = (uint32_t)args[2];
uint32_t v1 = (uint32_t)args[3];

v0 += (((v1 << 4) + 0x65766573) ^ (v1 + 0x22122122));
if (v0 != 0x93CD6340)
    return ALLOW;

v1 += (((v0 >> 5) + 0x6E6E6E6E) ^ (v0 + 0x22122122));
if (v1 != 0xB5F40D3F)
    return ALLOW;

if (syscall_number == 172 && (uint32_t)args[4] == 0x221221)
    return TRAP;

return ALLOW;
```

这实际上是一轮可逆的 TEA 风格运算。逆序先还原 `v1`，再还原 `v0`：

```python
MASK = 0xFFFFFFFF

out0 = 0x93CD6340
out1 = 0xB5F40D3F

v1 = (
    out1
    - (
        ((out0 >> 5) + 0x6E6E6E6E)
        ^ ((out0 + 0x22122122) & MASK)
    )
) & MASK

v0 = (
    out0
    - (
        (((v1 << 4) & MASK) + 0x65766573)
        ^ ((v1 + 0x22122122) & MASK)
    )
) & MASK

tail = v0.to_bytes(4, "little") + v1.to_bytes(4, "little")
print(hex(v0), hex(v1), tail.hex())
```

输出为：

```text
0x4d19d88c 0xef20af55 8cd8194d55af20ef
```

因此输入的后 8 字节已经确定为 `8c d8 19 4d 55 af 20 ef`。

触发 `TRAP` 后，程序进入 `SIGSYS` 处理函数。IDA 反编译时需要恢复 `ucontext_t` 类型。原 PDF 中“这是 amd64 架构”的文字是笔误；`ARCH_AARCH64`、`ARM64_SYSREG` 和以下寄存器布局都证明目标是 AArch64：

```c
struct sigcontext {
    uint64_t fault_address;
    uint64_t regs[31];
    uint64_t sp;
    uint64_t pc;
    uint64_t pstate;
    uint8_t reserved[4096] __attribute__((aligned(16)));
};
```

处理函数实际读取 `uc_mcontext.regs[0]` 和 `regs[2]`，也就是系统调用的第 1、3 个参数，恰好对应完整输入的前、后两个 8 字节。它将字符串 `Sevenlikeseccmop` 与输入后 8 字节循环异或，派生出 16 字节 AES 密钥：

```python
seed = b"Sevenlikeseccmop"
tail = bytes.fromhex("8c d8 19 4d 55 af 20 ef")
key = bytes(seed[i] ^ tail[i % 8] for i in range(16))
print(key.hex())
```

结果为：

```text
dfbd6f283bc34984e9ab7c2e36c24f9f
```

信号处理函数使用这个密钥对 16 字节输入执行 AES-ECB 加密，并与内置常量比较：

```text
b7 62 40 6a eb 70 b9 ed 81 71 db 9d ac 82 ff 94
```

所以直接进行 AES-ECB 解密即可恢复完整输入：

```python
from Crypto.Cipher import AES

key = bytes.fromhex("df bd 6f 28 3b c3 49 84 e9 ab 7c 2e 36 c2 4f 9f")
ciphertext = bytes.fromhex(
    "b7 62 40 6a eb 70 b9 ed 81 71 db 9d ac 82 ff 94"
)

plaintext = AES.new(key, AES.MODE_ECB).decrypt(ciphertext)
print(plaintext.hex())
```

输出：

```text
34ae7f8b605945878cd8194d55af20ef
```

按程序读取的 UUID 格式分组，flag 为：

```text
hgame{34ae7f8b-6059-4587-8cd8-194d55af20ef}
```

原 PDF 只给出 CyberChef 解密入口，没有直接写最终 flag；这里的结果由 BPF 常量和 AES 密文本地复算，并与[公开选手复盘](https://sh10rl.top/posts/hgame-week2-writeup/)交叉核对。

## 方法总结

本题把一次输入校验拆到三个位置：JNI 动态注册函数负责准备参数，seccomp-BPF 约束后 8 字节并触发 `SIGSYS`，信号处理函数再利用 AArch64 `ucontext_t` 中保存的寄存器完成 AES 校验。还原时必须按执行顺序逆推：先反汇编 BPF，逆一轮 TEA 得到后 8 字节；再用它与 `Sevenlikeseccmop` 异或得到 AES 密钥；最后解密内置密文得到完整 16 字节 UUID。决定性障碍是隐藏控制流与算法还原，因此本题归入 `reverse`。
