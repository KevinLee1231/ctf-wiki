---
type: technique
tags: [pwn, technique]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/format-string.md
updated: 2026-05-21
---

# Format String Exploitation

## 适用场景

内存破坏、沙箱逃逸、低级原语或 exploit chain 是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 出现 crash、越界读写、UAF、format string、heap metadata、seccomp 或 kernel primitive。
- 保护组合、libc、架构或 syscall 约束会影响路线。
- 需要 leak/write/control-flow hijack 串成最终能力。
- 题面或 raw 线索能落到这些关键词之一：Format String Basics、Argument Retargeting (Non-Positional %n Trick)、Blind Pwn (No Binary Provided)、Format String with Filter Bypass、Format String Canary + PIE Leak、freehook Overwrite via Format String (glibc < 2.34)、.rela.plt / .dynsym Patching、Format String for Game State Manipulation (UTCTF 2026)。

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
| Format String Basics | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Argument Retargeting (Non-Positional %n Trick) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Blind Pwn (No Binary Provided) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Format String with Filter Bypass | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Format String Canary + PIE Leak | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| freehook Overwrite via Format String (glibc < 2.34) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| .rela.plt / .dynsym Patching | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Format String for Game State Manipulation (UTCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Format String Saved EBP Overwrite for .bss Pivot (PlaidCTF 2015) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| argv[0] Overwrite for Stack Smash Info Leak (HITCON CTF 2015) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Format String .finiarray Loop for Multi-Stage Exploitation (Codegate 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| printfchk Bypass with Sequential %p (VolgaCTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Leak + GOT Overwrite in Single printf Call (picoCTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Objective-C %@ Format Specifier Exploitation (SHA2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| strlen Integer Truncation Bypass (ASIS CTF Finals 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| printffunctiontable Overwrite via Buffer Overflow (34C3 CTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| scanf Format String on Stack Overwrite (TUCTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Format String Exploit Through ROT13 Encoding (SunshineCTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Format String Through Input Transformation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Format String .finiarray Loop for Multi-Stage Exploitation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [oob-jit-parser-primitives-family.md](oob-jit-parser-primitives-family.md)
- [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md)
- [emulator-float-and-hash-exploits.md](emulator-float-and-hash-exploits.md)
- [heap-fsop-file-structure-attacks.md](heap-fsop-file-structure-attacks.md)
- [heap-houses-unlink-and-tcache.md](heap-houses-unlink-and-tcache.md)
- [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md)

## 原始资料

- [format-string.md](../raw/pwn/format-string.md)
