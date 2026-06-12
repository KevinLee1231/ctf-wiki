---
type: technique
tags: [pwn, technique]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/overflow-basics.md
updated: 2026-05-21
---

# Overflow Basics

## 适用场景

内存破坏、沙箱逃逸、低级原语或 exploit chain 是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 出现 crash、越界读写、UAF、format string、heap metadata、seccomp 或 kernel primitive。
- 保护组合、libc、架构或 syscall 约束会影响路线。
- 需要 leak/write/control-flow hijack 串成最终能力。
- 题面或 raw 线索能落到这些关键词之一：Stack Buffer Overflow、ret2win with Parameter (Magic Value Check)、Stack Alignment (16-byte Requirement)、Offset Calculation from Disassembly、Input Filtering (memmem checks)、Finding Gadgets、Hidden Gadgets in CMP Immediates、Struct Pointer Overwrite (Heap Menu Challenges)。

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
| Stack Buffer Overflow | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| ret2win with Parameter (Magic Value Check) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Stack Alignment (16-byte Requirement) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Offset Calculation from Disassembly | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Input Filtering (memmem checks) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Finding Gadgets | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Hidden Gadgets in CMP Immediates | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Struct Pointer Overwrite (Heap Menu Challenges) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Signed Integer Bypass (Negative Quantity) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Canary-Aware Partial Overflow | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| OOB Read via Stride/Rate Leak (DiceCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Stack Canary Byte-by-Byte Brute Force on Forking Servers | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Global Buffer Overflow (CSV Injection) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Protocol Length Field Stack Bleeding (EKOPARTY CTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Protocol Stack Bleeding | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Parser Stack Overflow via Unchecked memcpy Length (MetaCTF Flash 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Parser Stack Overflow (Unchecked memcpy) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Stack Canary Null-Byte Overwrite Leak (CSAW 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Empty-Token strncmp(n=0) MAC Bypass (UCSB iCTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Return Address LSB Overwrite + read() Chaining (TUCTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Canary Trailing-Byte Leak via Padding One Byte Past Null (hxp 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Index-Only Bounds Check + Stride OOB Write (P.W.N. CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Signed Index Negative OOB to Preceding GOT (P.W.N. CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| PIE Same-Page Function Pivot via Single-Byte Overwrite (P.W.N. CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

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

- [overflow-basics.md](../raw/pwn/overflow-basics.md)
