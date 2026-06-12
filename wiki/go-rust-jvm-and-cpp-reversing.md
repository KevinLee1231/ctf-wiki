---
type: technique
tags: [reverse, technique]
skills: [ctf-reverse]
raw:
  - ../raw/reverse/go-rust-jvm-and-cpp-reversing.md
updated: 2026-05-21
---

# Go, Rust, JVM and C++ Reversing

## 适用场景

需要理解二进制、脚本、字节码、壳、VM、固件或混淆逻辑，再恢复算法或输入。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 附件是 ELF/PE/APK/WASM/pyc/固件/脚本，或存在壳、SMC、自定义 VM。
- flag 校验藏在运行时生成代码、解密字符串、解释器或 native 扩展中。
- 静态字符串不足，需要交叉引用、动态断点或 trace。
- 题面或 raw 线索能落到这些关键词之一：Go Binary Reversing、Recognition、Symbol Recovery、Go Memory Layout、Goroutine and Concurrency Analysis、Common Go Patterns in Decompilation、Go Binary Reversing Workflow、Go Binary UUID Patching for C2 Client Enumeration (BSidesSF 2026)。

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
| Go Binary Reversing | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Recognition | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Symbol Recovery | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Go Memory Layout | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Goroutine and Concurrency Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Common Go Patterns in Decompilation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Go Binary Reversing Workflow | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Go Binary UUID Patching for C2 Client Enumeration (BSidesSF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Go Binary UUID Patching for C2 Enumeration | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Rust Binary Reversing | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Rust Recognition | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Symbol Demangling | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Common Rust Patterns in Decompilation | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Rust-Specific Analysis Tools | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Swift Binary Reversing | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Swift / Kotlin Binary Reversing | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Kotlin / JVM Binary Reversing | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| JVM Bytecode (Android/Server) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Kotlin/Native | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| D Language Binary Reversing (CSAW CTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| D Language Binary Reversing | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Haskell Binary Reversing via STG Closures and hsdecomp (hxp CTF 2017, Codegate 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Haskell Binary RE via GHC CMM Intermediate Language (N1CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| C++ Binary Reversing (速查) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

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

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [HGAME2026-衔尾蛇-wp](../raw/reverse/HGAME2026-衔尾蛇-wp.md) | Spring Boot/JAR 题要先区分表面业务实现和运行时替换后的真实校验引擎。 |
| [HGAME2026-pvz-wp](../raw/reverse/HGAME2026-pvz-wp.md) | Java 游戏题可以把资源提示、业务类和小范围 hash 状态爆破结合，而不是只搜字符串。 |
| [ACTF2026-calc-my-point-wp](../raw/reverse/ACTF2026-calc-my-point-wp.md) | Rust/GMP/大整数表达式题先还原语言运行时对象和数值约束，再做 CRT 或代数求解。 |
| [D3CTF2022-d3thon-wp](../raw/reverse/D3CTF2022-d3thon-wp.md) | Cython/Python 扩展题应恢复 Python 调用边界和 native 校验函数，不要只看反编译 Python。 |

## 原始资料

- [go-rust-jvm-and-cpp-reversing.md](../raw/reverse/go-rust-jvm-and-cpp-reversing.md)
