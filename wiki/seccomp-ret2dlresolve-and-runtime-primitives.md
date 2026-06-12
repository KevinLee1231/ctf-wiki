---
type: technique
tags: [pwn, technique]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/seccomp-ret2dlresolve-and-runtime-primitives.md
updated: 2026-05-21
---

# Seccomp, ret2dlresolve and Runtime Primitives

## 适用场景

内存破坏、沙箱逃逸、低级原语或 exploit chain 是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 出现 crash、越界读写、UAF、format string、heap metadata、seccomp 或 kernel primitive。
- 保护组合、libc、架构或 syscall 约束会影响路线。
- 需要 leak/write/control-flow hijack 串成最终能力。
- 题面或 raw 线索能落到这些关键词之一：Seccomp Advanced Techniques、openat2 Bypass (New Age Pattern)、Conditional Buffer Address Restrictions、Shellcode Construction Without Relocations (pwntools)、Seccomp Analysis from Disassembly、rdx Control in ROP Chains、Use-After-Free (UAF) Exploitation、JIT Compilation Exploits。

## 最小证据

- 已完成主方向判断，并确认本页技巧比相邻技巧更能解释当前证据。
- 至少有一个可复现输入、输出、文件结构、数学关系、协议行为或运行时状态。
- 能指出 raw 案例中哪一个变体与当前题最接近，以及不同点在哪里。

## 解法骨架

1. 稳定复现 bug。
2. 量化 read/write/control primitive。
3. 按保护选择利用路线。
4. 脚本化 exploit 并处理远程差异。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| Seccomp Advanced Techniques | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| openat2 Bypass (New Age Pattern) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Conditional Buffer Address Restrictions | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Shellcode Construction Without Relocations (pwntools) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Seccomp Analysis from Disassembly | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| rdx Control in ROP Chains | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Use-After-Free (UAF) Exploitation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| JIT Compilation Exploits | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Esoteric Language GOT Overwrite | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Heap Overlap via Base Conversion | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Tree Data Structure Stack Underallocation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| ret2dlresolve | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Kernel Exploitation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| 9-Byte test+je Timing Leak (hxp 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RtlCaptureContext Deterministic Windows Stack Leak (Insomnihack 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| IEEE 754 Double-as-Shellcode via Exponent Fixing (Kaspersky 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| PIE Bypass via Consistent glibc Load Base 0x56555000 (TAMUctf 2019) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| 运行时短案例技巧族：类型、索引、格式化与 GC 原语 | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Type Confusion in Interpreters | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Off-by-One Index / Size Corruption | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Double win() Call | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| DNS Record Buffer Overflow | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| ASAN Shadow Memory Exploitation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Format String with RWX .finiarray Hijack | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [oob-jit-parser-primitives-family.md](oob-jit-parser-primitives-family.md)
- [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md)
- [emulator-float-and-hash-exploits.md](emulator-float-and-hash-exploits.md)
- [format-string.md](format-string.md)
- [heap-fsop-file-structure-attacks.md](heap-fsop-file-structure-attacks.md)
- [heap-houses-unlink-and-tcache.md](heap-houses-unlink-and-tcache.md)

## 原始资料

- [seccomp-ret2dlresolve-and-runtime-primitives.md](../raw/pwn/seccomp-ret2dlresolve-and-runtime-primitives.md)
