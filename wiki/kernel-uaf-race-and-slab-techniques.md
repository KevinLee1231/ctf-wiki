---
type: technique
tags: [pwn, technique]
skills: [ctf-pwn]
raw:
  - ../raw/pwn/kernel-uaf-race-and-slab-techniques.md
updated: 2026-05-21
---

# Kernel UAF, Race and SLAB Techniques

## 适用场景

内存破坏、沙箱逃逸、低级原语或 exploit chain 是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 出现 crash、越界读写、UAF、format string、heap metadata、seccomp 或 kernel primitive。
- 保护组合、libc、架构或 syscall 约束会影响路线。
- 需要 leak/write/control-flow hijack 串成最终能力。
- 题面或 raw 线索能落到这些关键词之一：ttystruct RIP Hijack and kROP、kROP via Fake Vtable on ttystruct、AAW via ioctl Register Control、userfaultfd Race Stabilization、Alternative Race Techniques (uffd Disabled)、SLUB Allocator Internals、Freelist Pointer Hardening、Freelist Obfuscation (CONFIGSLABFREELISTHARDEN)。

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
| ttystruct RIP Hijack and kROP | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| kROP via Fake Vtable on ttystruct | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| AAW via ioctl Register Control | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| userfaultfd Race Stabilization | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Alternative Race Techniques (uffd Disabled) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| SLUB Allocator Internals | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Freelist Pointer Hardening | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Freelist Obfuscation (CONFIGSLABFREELISTHARDEN) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Leak via Kernel Panic | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Race Window Extension via MADVDONTNEED + mprotect (DiceCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Cross-Cache Attack via CPU-Split Strategy (DiceCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| PTE Overlap Primitive for File Write (DiceCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Kernel addrlimit Bypass via Failed File Open (Midnight Sun CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Kernel Exploitation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

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

- [kernel-uaf-race-and-slab-techniques.md](../raw/pwn/kernel-uaf-race-and-slab-techniques.md)
