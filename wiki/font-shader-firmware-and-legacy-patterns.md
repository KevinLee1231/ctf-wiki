---
type: technique
tags: [reverse, technique]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/font-shader-firmware-and-legacy-patterns.md
updated: 2026-05-21
---

# Font, Shader, Firmware and Legacy Patterns

## 适用场景

需要理解二进制、脚本、字节码、壳、VM、固件或混淆逻辑，再恢复算法或输入。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 附件是 ELF/PE/APK/WASM/pyc/固件/脚本，或存在壳、SMC、自定义 VM。
- flag 校验藏在运行时生成代码、解密字符串、解释器或 native 扩展中。
- 静态字符串不足，需要交叉引用、动态断点或 trace。
- 题面或 raw 线索能落到这些关键词之一：Z3 for Single-Line Python Boolean Circuit (BearCatCTF 2026)、Z3 for Single-Line Python Boolean Circuit、Sliding Window Popcount Differential Propagation (BearCatCTF 2026)、Sliding Window Popcount Differential Propagation、Morse Code from Keyboard LEDs via ioctl (PlaidCTF 2013)、C++ Destructor-Hidden Validation (Defcamp 2015)、Syscall Side-Effect Memory Corruption (Hack.lu 2015)、MFC Dialog Event Handler Location (WhiteHat 2015)。

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
| Z3 for Single-Line Python Boolean Circuit (BearCatCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Z3 for Single-Line Python Boolean Circuit | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Sliding Window Popcount Differential Propagation (BearCatCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Sliding Window Popcount Differential Propagation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Morse Code from Keyboard LEDs via ioctl (PlaidCTF 2013) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| C++ Destructor-Hidden Validation (Defcamp 2015) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Syscall Side-Effect Memory Corruption (Hack.lu 2015) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| MFC Dialog Event Handler Location (WhiteHat 2015) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| VM Sequential Key-Chain Brute-Force (Midnight Flag 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Burrows-Wheeler Transform Inversion without Terminator (ASIS CTF Finals 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Burrows-Wheeler Transform Inversion | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| OpenType Font Ligature Exploitation for Hidden Messages (Hack The Vote 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Font Ligature Exploitation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| GLSL Shader VM with Self-Modifying Code (ApoorvCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| GLSL Shader VM with Self-Modifying Code | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Instruction Counter as Cryptographic State (MetaCTF Flash 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Instruction Counter as Cryptographic State | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Thread Race Condition with Signed Integer Overflow (Codegate 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Thread Race Signed Integer Overflow | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| ESP32/Xtensa Firmware Reversing with ROM Symbol Map (Insomni'hack 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| ESP32/Xtensa Firmware Reversing | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Batch Crackme Automation via objdump Pattern Extraction (DEF CON 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Batch Crackme Automation via objdump | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Fork + Pipe + Dead Branch Anti-Analysis (RCTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

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

- [font-shader-firmware-and-legacy-patterns.md](../raw/reverse/font-shader-firmware-and-legacy-patterns.md)
