---
type: technique
tags: [pwn, technique]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/heap-fsop-file-structure-attacks.md
updated: 2026-05-21
---

# Heap FSOP and FILE Structure Attacks

## 适用场景

内存破坏、沙箱逃逸、低级原语或 exploit chain 是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 出现 crash、越界读写、UAF、format string、heap metadata、seccomp 或 kernel primitive。
- 保护组合、libc、架构或 syscall 约束会影响路线。
- 需要 leak/write/control-flow hijack 串成最终能力。
- 题面或 raw 线索能落到这些关键词之一：Fastbin stdout Vtable Two-Stage Hijack for PIE + Full RELRO (ASIS CTF 2017)、IObufbase Null Byte Overwrite for stdin Hijack (Tokyo Westerns 2017)、glibc 2.24+ IOFILE Vtable Validation Bypass (HITCON 2017)、Unsorted Bin Attack on stdin IObufend (HITCON 2017)、Unsorted Bin Corruption via mp Structure (HITCON 2017)、realloc(ptr, 0) as free() for UAF (AceBear 2018)、Single-Byte Reference Counter Wraparound to UAF (WhiteHat Grand Prix 2018)。

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
| Fastbin stdout Vtable Two-Stage Hijack for PIE + Full RELRO (ASIS CTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| IObufbase Null Byte Overwrite for stdin Hijack (Tokyo Westerns 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| glibc 2.24+ IOFILE Vtable Validation Bypass (HITCON 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Unsorted Bin Attack on stdin IObufend (HITCON 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Unsorted Bin Corruption via mp Structure (HITCON 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| realloc(ptr, 0) as free() for UAF (AceBear 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Single-Byte Reference Counter Wraparound to UAF (WhiteHat Grand Prix 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [cross-primitive-escape-and-hybrid-exploit-map.md](cross-primitive-escape-and-hybrid-exploit-map.md)
- [emulator-float-and-hash-exploits.md](emulator-float-and-hash-exploits.md)
- [format-string.md](format-string.md)
- [heap-houses-unlink-and-tcache.md](heap-houses-unlink-and-tcache.md)
- [heap-uaf-tcache-and-custom-allocator.md](heap-uaf-tcache-and-custom-allocator.md)

## 原始资料

- [heap-fsop-file-structure-attacks.md](../raw/pwn/heap-fsop-file-structure-attacks.md)
