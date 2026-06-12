---
type: technique
tags: [reverse, technique]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/self-decrypting-strings-and-lattice-patterns.md
updated: 2026-05-21
---

# Self-Decrypting, String and Lattice Patterns

## 适用场景

需要理解二进制、脚本、字节码、壳、VM、固件或混淆逻辑，再恢复算法或输入。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 附件是 ELF/PE/APK/WASM/pyc/固件/脚本，或存在壳、SMC、自定义 VM。
- flag 校验藏在运行时生成代码、解密字符串、解释器或 native 扩展中。
- 静态字符串不足，需要交叉引用、动态断点或 trace。
- 题面或 raw 线索能落到这些关键词之一：Multi-Layer Self-Decrypting Binary (DiceCTF 2026)、Multi-Layer Self-Decrypting Binary、Embedded ZIP + XOR License Decryption (MetaCTF 2026)、Embedded ZIP + XOR License Decryption、Stack String Deobfuscation from .rodata XOR Blob (Nullcon 2026)、Stack String Deobfuscation (.rodata XOR Blob)、Prefix Hash Brute-Force (Nullcon 2026)、Prefix Hash Brute-Force。

## 最小证据

- 已完成主方向判断，并确认本页技巧比相邻技巧更能解释当前证据。
- 至少有一个可复现输入、输出、文件结构、数学关系、协议行为或运行时状态。
- 能指出 raw 案例中哪一个变体与当前题最接近，以及不同点在哪里。

## 解法骨架

1. 先做载体、字符串、导入和入口函数首检。
2. 定位真实校验、解密、分发或比较点。
3. 把复杂逻辑降维成约束、解密脚本或 oracle。
4. 用 solver / forward check 验证输入。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| Multi-Layer Self-Decrypting Binary (DiceCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Multi-Layer Self-Decrypting Binary | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Embedded ZIP + XOR License Decryption (MetaCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Embedded ZIP + XOR License Decryption | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Stack String Deobfuscation from .rodata XOR Blob (Nullcon 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Stack String Deobfuscation (.rodata XOR Blob) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Prefix Hash Brute-Force (Nullcon 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Prefix Hash Brute-Force | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| CVP/LLL Lattice for Constrained Integer Validation (HTB ShadowLabyrinth) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| CVP/LLL Lattice for Constrained Integer Validation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Decision Tree Function Obfuscation (HTB WonderSMS) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Decision Tree Function Obfuscation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| GF(2^8) Gaussian Elimination for Flag Recovery (ApoorvCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| GF(2^8) Gaussian Elimination for Flag Recovery | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| ROP Chain Obfuscation in Modified Binary (PlaidCTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Common Encryption Patterns | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)
- [anti-analysis.md](anti-analysis.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)
- [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)
- [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md)

## 原始资料

- [self-decrypting-strings-and-lattice-patterns.md](../raw/reverse/self-decrypting-strings-and-lattice-patterns.md)
