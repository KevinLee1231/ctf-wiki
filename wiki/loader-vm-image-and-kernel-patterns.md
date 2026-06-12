---
type: technique
tags: [reverse, technique]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/loader-vm-image-and-kernel-patterns.md
updated: 2026-05-21
---

# Loader, VM, Image and Kernel Patterns

## 适用场景

需要理解二进制、脚本、字节码、壳、VM、固件或混淆逻辑，再恢复算法或输入。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 附件是 ELF/PE/APK/WASM/pyc/固件/脚本，或存在壳、SMC、自定义 VM。
- flag 校验藏在运行时生成代码、解密字符串、解释器或 native 扩展中。
- 静态字符串不足，需要交叉引用、动态断点或 trace。
- 题面或 raw 线索能落到这些关键词之一：Hidden Emulator Opcodes + LDPRELOAD Key Extraction (0xFun 2026)、Spectre-RSB SPN Cipher — Static Parameter Extraction (0xFun 2026)、Image XOR Mask Recovery via Smoothness (VuwCTF 2025)、Shellcode in Data Section via mmap RWX (VuwCTF 2025)、Recursive execve Subtraction (VuwCTF 2025)、Byte-at-a-Time Block Cipher Attack (UTCTF 2024)、Mathematical Convergence Bitmap (EHAX 2026)、Mathematical Convergence Bitmap。

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
| Hidden Emulator Opcodes + LDPRELOAD Key Extraction (0xFun 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Spectre-RSB SPN Cipher — Static Parameter Extraction (0xFun 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Image XOR Mask Recovery via Smoothness (VuwCTF 2025) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Shellcode in Data Section via mmap RWX (VuwCTF 2025) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Recursive execve Subtraction (VuwCTF 2025) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Byte-at-a-Time Block Cipher Attack (UTCTF 2024) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Mathematical Convergence Bitmap (EHAX 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Mathematical Convergence Bitmap | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Windows PE XOR Bitmap Extraction + OCR (srdnlenCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Two-Stage Loader: RC4 Gate + VM Constraints (srdnlenCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| GBA ROM VM Hash Inversion via Meet-in-the-Middle (srdnlenCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Sprague-Grundy Game Theory Binary (DiceCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Sprague-Grundy Game Theory Binary | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Kernel Module Maze Solving (DiceCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Kernel Module Maze Solving | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Multi-Threaded VM with Channel Synchronization (DiceCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Multi-Threaded VM with Channels | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Backdoored Shared Library Detection via String Diffing (Hack.lu CTF 2012) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Backdoored Shared Library Detection | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Custom binfmt Kernel Module with RC4 Flat Binaries (BSidesSF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Custom binfmt Kernel Module with RC4 Flat Binaries | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Hash-Resolved Imports / No-Import Ransomware (BSidesSF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Hash-Resolved Imports / No-Import Ransomware | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| ELF Section Header Corruption for Anti-Analysis (BSidesSF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

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

- [loader-vm-image-and-kernel-patterns.md](../raw/reverse/loader-vm-image-and-kernel-patterns.md)
