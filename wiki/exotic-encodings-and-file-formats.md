---
type: technique
tags: [misc, technique]
skills: [ctf-misc]
raw:
  - ../raw/misc/exotic-encodings-and-file-formats.md
updated: 2026-05-21
---

# Exotic Encodings and File Formats

## 适用场景

编码、jail、RF/音频、esolang、约束求解或跨方向轻量技巧是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 题目没有明确落入更具体主线。
- 输入输出像编码、语法限制、逻辑谜题、交互游戏或特殊格式。
- 可以用小脚本快速验证候选模式。
- 题面或 raw 线索能落到这些关键词之一：Verilog/HDL、Gray Code Cyclic Encoding (EHAX 2026)、Binary Tree Key Encoding、RTF Custom Tag Data Extraction (VolgaCTF 2013)、SMS PDU Decoding and Reassembly (RuCTF 2013)、Automated Multi-Encoding Sequential Solver (HackIM 2016)、RFC 4042 UTF-9 Decoding (SECCON 2015)、Pixel Color Binary Encoding (Break In 2016)。

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
| Verilog/HDL | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Gray Code Cyclic Encoding (EHAX 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Binary Tree Key Encoding | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RTF Custom Tag Data Extraction (VolgaCTF 2013) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| SMS PDU Decoding and Reassembly (RuCTF 2013) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Automated Multi-Encoding Sequential Solver (HackIM 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RFC 4042 UTF-9 Decoding (SECCON 2015) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Pixel Color Binary Encoding (Break In 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Hexadecimal Sudoku + QR Assembly (BSidesSF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| TOPKEK Binary Encoding (Hack The Vote 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| MaxiCode 2D Barcode Decoding (CSAW CTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| DTMF Audio with Multi-Tap Phone Keypad Decoding (h4ckc0n 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Music Note Interval Steganography (DefCamp 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Ruby Array#unpack Buffer Under-Read CVE-2018-8778 (Codegate 2019) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Binary Grid Text to QR Image + XOR Key (Pragyan CTF 2019) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [bashjails.md](bashjails.md)
- [dns.md](dns.md)
- [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md)
- [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md)
- [game-state-websocket-and-wasm.md](game-state-websocket-and-wasm.md)

## 原始资料

- [exotic-encodings-and-file-formats.md](../raw/misc/exotic-encodings-and-file-formats.md)
