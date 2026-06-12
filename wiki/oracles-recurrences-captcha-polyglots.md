---
type: technique
tags: [misc, technique]
skills: [ctf-misc]
raw:
  - ../raw/misc/oracles-recurrences-captcha-polyglots.md
updated: 2026-05-21
---

# Oracles, Recurrences, CAPTCHA and Polyglots

## 适用场景

编码、jail、RF/音频、esolang、约束求解或跨方向轻量技巧是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 题目没有明确落入更具体主线。
- 输入输出像编码、语法限制、逻辑谜题、交互游戏或特殊格式。
- 可以用小脚本快速验证候选模式。
- 题面或 raw 线索能落到这些关键词之一：XSLT as Turing-Complete VM for Binary Search (35C3 2018)、JavaScript MAXSAFEINTEGER Successor Equality (35C3 2018)、Binary Search Oracle in Comparison-Only DSL (35C3 2018)、Blind SQLi via Script-Engine Timeout Error (35C3 2018)、OEIS Sequence Lookup Automation for Recurrence Puzzles (X-MAS CTF 2018)、QR Code Reassembly from Format-String Structural Constraints (Square CTF 2018)、Matrix Exponentiation for Fibonacci-Like Recurrence (Pwn2Win 2018)、Tribonacci Recurrence for Frog Jump Counting (FireShell 2019)。

## 最小证据

- 已完成主方向判断，并确认本页技巧比相邻技巧更能解释当前证据。
- 至少有一个可复现输入、输出、文件结构、数学关系、协议行为或运行时状态。
- 能指出 raw 案例中哪一个变体与当前题最接近，以及不同点在哪里。

## 解法骨架

1. 排除更具体专项方向。
2. 做格式、字符集、频率、长度和交互规律检查。
3. 把问题转成枚举、约束或模拟。
4. 用最短脚本验证并复现。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| XSLT as Turing-Complete VM for Binary Search (35C3 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| JavaScript MAXSAFEINTEGER Successor Equality (35C3 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Binary Search Oracle in Comparison-Only DSL (35C3 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Blind SQLi via Script-Engine Timeout Error (35C3 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| OEIS Sequence Lookup Automation for Recurrence Puzzles (X-MAS CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| QR Code Reassembly from Format-String Structural Constraints (Square CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Matrix Exponentiation for Fibonacci-Like Recurrence (Pwn2Win 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Tribonacci Recurrence for Frog Jump Counting (FireShell 2019) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Selenium + Tesseract for Dynamic Font CAPTCHA (Square CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Brainfuck Decodes Piet Image URL — Multi-Layer Polyglot (RITSEC 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Bytebeat Synth Code Recognition for Hidden Audio (RITSEC 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| SHA-256 Length Extension Attack | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Levenshtein Distance Oracle Attack (SunshineCTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [bashjails.md](bashjails.md)
- [dns.md](dns.md)
- [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md)
- [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md)
- [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md)

## 原始资料

- [oracles-recurrences-captcha-polyglots.md](../raw/misc/oracles-recurrences-captcha-polyglots.md)
