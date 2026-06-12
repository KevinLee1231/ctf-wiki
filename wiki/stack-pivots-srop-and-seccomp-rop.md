---
type: technique
tags: [pwn, technique]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/stack-pivots-srop-and-seccomp-rop.md
updated: 2026-05-21
---

# Stack Pivots, SROP and Seccomp ROP

## 适用场景

内存破坏、沙箱逃逸、低级原语或 exploit chain 是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 出现 crash、越界读写、UAF、format string、heap metadata、seccomp 或 kernel primitive。
- 保护组合、libc、架构或 syscall 约束会影响路线。
- 需要 leak/write/control-flow hijack 串成最终能力。
- 题面或 raw 线索能落到这些关键词之一：Double Stack Pivot to BSS via leave;ret (Midnightflag 2026)、Double Stack Pivot to BSS via leave;ret、SROP with UTF-8 Payload Constraints (DiceCTF 2026)、Seccomp Bypass、RETF Architecture Switch for Seccomp Bypass (Midnightflag 2026)、RETF Architecture Switch for Seccomp Bypass、Stack Shellcode with Input Reversal、.finiarray Hijack。

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
| Double Stack Pivot to BSS via leave;ret (Midnightflag 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Double Stack Pivot to BSS via leave;ret | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| SROP with UTF-8 Payload Constraints (DiceCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Seccomp Bypass | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RETF Architecture Switch for Seccomp Bypass (Midnightflag 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RETF Architecture Switch for Seccomp Bypass | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Stack Shellcode with Input Reversal | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| .finiarray Hijack | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| pwntools Template | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Automated Offset Finding via Corefile (Crypto-Cat) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| ret2vdso — Using Kernel vDSO Gadgets (HTB Nowhere to go) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Vsyscall ROP for PIE Bypass (Hack.lu 2015) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| x32 ABI Syscall Number Aliasing for Seccomp Bypass (BCTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Time-Based Blind Shellcode When write() Blocked (DEF CON 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| JIT-ROP: Scan for syscall Byte in Leaked libc Function (Codegate 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| ret2dlresolve 64-bit (Codegate 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Prime-Only ROP via Goldbach Decomposition (PlaidCTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Imperfect-Gadget Stack Pivot (RITSEC 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| finiarray Double-Entry Staged ROP (Insomnihack 2019) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| ret2libc via Statically-Linked libc + Embedded /bin/sh String (TAMUctf 2019) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Useful Commands | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| SROP with UTF-8 Constraints | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md)
- [emulator-float-and-hash-exploits.md](emulator-float-and-hash-exploits.md)
- [format-string.md](format-string.md)
- [heap-fsop-file-structure-attacks.md](heap-fsop-file-structure-attacks.md)
- [heap-houses-unlink-and-tcache.md](heap-houses-unlink-and-tcache.md)

## 原始资料

- [stack-pivots-srop-and-seccomp-rop.md](../raw/pwn/stack-pivots-srop-and-seccomp-rop.md)
