---
type: technique
tags: [reverse, technique]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/runtime-patching-oracles-and-tracing.md
updated: 2026-05-21
---

# Runtime Patching, Oracles and Tracing

## 适用场景

需要理解二进制、脚本、字节码、壳、VM、固件或混淆逻辑，再恢复算法或输入。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 附件是 ELF/PE/APK/WASM/pyc/固件/脚本，或存在壳、SMC、自定义 VM。
- flag 校验藏在运行时生成代码、解密字符串、解释器或 native 扩展中。
- 静态字符串不足，需要交叉引用、动态断点或 trace。
- 题面或 raw 线索能落到这些关键词之一：Malware Anti-Analysis Bypass via Patching、Multi-Stage Shellcode Loaders、Timing Side-Channel Attack、Multi-Thread Anti-Debug with Decoy + Signal Handler Mixed Boolean-Arithmetic (ApoorvCTF…、INT3 Patch + Coredump Brute-Force Oracle (Pwn2Win 2016)、INT3 Patch + Coredump Brute-Force Oracle、Signal Handler Chain + LDPRELOAD Oracle (Nuit du Hack 2016)、printf Format String VM Decompilation to Z3 (SECCON 2017)。

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
| Malware Anti-Analysis Bypass via Patching | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Multi-Stage Shellcode Loaders | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Timing Side-Channel Attack | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Multi-Thread Anti-Debug with Decoy + Signal Handler Mixed Boolean-Arithmetic (ApoorvCTF… | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| INT3 Patch + Coredump Brute-Force Oracle (Pwn2Win 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| INT3 Patch + Coredump Brute-Force Oracle | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Signal Handler Chain + LDPRELOAD Oracle (Nuit du Hack 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| printf Format String VM Decompilation to Z3 (SECCON 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Quadtree Recursive Image Format Parser (Google CTF Quals 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [vm-obfuscation-transform-family.md](vm-obfuscation-transform-family.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)
- [vmp-client-server-smc-rc4-recovery.md](vmp-client-server-smc-rc4-recovery.md)
- [android-games-hardware-and-runtime-platforms.md](android-games-hardware-and-runtime-platforms.md)
- [anti-analysis.md](anti-analysis.md)
- [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)

## 原始资料

- [runtime-patching-oracles-and-tracing.md](../raw/reverse/runtime-patching-oracles-and-tracing.md)
