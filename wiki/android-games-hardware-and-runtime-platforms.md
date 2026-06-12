---
type: technique
tags: [reverse, technique]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/android-games-hardware-and-runtime-platforms.md
updated: 2026-05-21
---

# Android, Games, Hardware and Runtime Platforms

## 适用场景

需要理解二进制、脚本、字节码、壳、VM、固件或混淆逻辑，再恢复算法或输入。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 附件是 ELF/PE/APK/WASM/pyc/固件/脚本，或存在壳、SMC、自定义 VM。
- flag 校验藏在运行时生成代码、解密字符串、解释器或 native 扩展中。
- 静态字符串不足，需要交叉引用、动态断点或 trace。
- 题面或 raw 线索能落到这些关键词之一：Roblox Place File Analysis、Godot Game Asset Extraction、Rust serdejson Schema Recovery、Android JNI RegisterNatives Obfuscation (HTB WonderSMS)、Android JNI RegisterNatives Obfuscation、Android DEX Runtime Bytecode Patching via /proc/self/maps (Google CTF 2017)、Android DEX Runtime Bytecode Patching、Android Native .so Loading Bypass in New Project (Codegate CTF 2018)。

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
| Roblox Place File Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Godot Game Asset Extraction | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Rust serdejson Schema Recovery | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Android JNI RegisterNatives Obfuscation (HTB WonderSMS) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Android JNI RegisterNatives Obfuscation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Android DEX Runtime Bytecode Patching via /proc/self/maps (Google CTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Android DEX Runtime Bytecode Patching | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Android Native .so Loading Bypass in New Project (Codegate CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Frida Firebase Cloud Functions Bypass (BSidesSF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Frida Firebase Cloud Functions Bypass | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Verilog/Hardware Reverse Engineering (srdnlenCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Verilog/Hardware RE | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Prefix-by-Prefix Hash Reversal (Nullcon 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Ruby/Perl Polyglot Constraint Satisfaction (BearCatCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Ruby/Perl Polyglot Constraint Satisfaction | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Electron App + Native Binary Reversing (RootAccess2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Node.js npm Package Runtime Introspection (RootAccess2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Frida Android Certificate Pinning Bypass (h1702ctf 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Android Anti-Debug: TracerPid, su Binary, System Properties (h1702ctf 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Android Log-Based Key Extraction (HackIT 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Native JNI Key Extraction via Memory Dump and Smali Patching (HackIT 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| IBM AS/400 SAVF File EBCDIC Decoding (EKOPARTY 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Intel SGX Enclave Reverse Engineering (Pwn2Win 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Glulx Interactive Fiction Bytecode Matrix Validation (PlaidCTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [anti-analysis.md](anti-analysis.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)
- [disassemblers-debuggers-and-basic-tools.md](disassemblers-debuggers-and-basic-tools.md)
- [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md)
- [font-shader-firmware-and-legacy-patterns.md](font-shader-firmware-and-legacy-patterns.md)

## 原始资料

- [android-games-hardware-and-runtime-platforms.md](../raw/reverse/android-games-hardware-and-runtime-platforms.md)
