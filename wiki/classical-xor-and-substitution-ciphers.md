---
type: technique
tags: [crypto, technique]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/classical-xor-and-substitution-ciphers.md
updated: 2026-05-21
---

# Classical XOR and Substitution Ciphers

## 适用场景

密钥恢复、原语误用、oracle、随机数、签名或协议假设是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 题目给出密文、nonce、签名、模数、oracle、PRNG 输出或自定义协议。
- 存在重复 nonce、弱随机、错误回显、数学结构或可查询接口。
- 源码能复现加密、签名、哈希或验证流程。
- 题面或 raw 线索能落到这些关键词之一：Vigenere Cipher、Kasiski Examination for Key Length、Atbash Cipher、Polybius Square Cipher (Qiwi-Infosec 2016)、Substitution Cipher with Rotating Wheel、XOR Variants、Multi-Byte XOR Key Recovery via Frequency Analysis、Cascade XOR (First-Byte Brute Force)。

## 最小证据

- 已完成主方向判断，并确认本页技巧比相邻技巧更能解释当前证据。
- 至少有一个可复现输入、输出、文件结构、数学关系、协议行为或运行时状态。
- 能指出 raw 案例中哪一个变体与当前题最接近，以及不同点在哪里。

## 解法骨架

1. 列清 known / unknown / goal。
2. 复现原语和约束。
3. 选择一个最匹配攻击族做最小验证。
4. 用解出的结果做正向复算。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| Vigenere Cipher | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Kasiski Examination for Key Length | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Atbash Cipher | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Polybius Square Cipher (Qiwi-Infosec 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Substitution Cipher with Rotating Wheel | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| XOR Variants | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Multi-Byte XOR Key Recovery via Frequency Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Cascade XOR (First-Byte Brute Force) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| XOR with Rotation: Power-of-2 Bit Isolation (Pragyan 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Weak XOR Verification Brute Force (Pragyan 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Deterministic OTP with Load-Balanced Backends (Pragyan 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| OTP Key Reuse / Many-Time Pad XOR (BYPASS CTF 2025) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Book Cipher | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Variable-Length Homophonic Substitution (ASIS CTF Finals 2013) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Grid Permutation Cipher Keyspace Reduction (BSidesSF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Image-Based Caesar Shift Ciphers (BSidesSF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Variant A — Vertical Strip Shift (caesar1) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Variant B — Horizontal Shift with ASCII Encoding (caesar2) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| XOR Key Recovery via File Format Headers (MetaCTF Flash 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| 3D Vigenere Palindrome Symmetry Key Recovery (SECCON 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Nihilist Cipher Double-Crib Key Recovery (Security Fest CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| 16-Byte XOR Block Cipher Structural Reversal (h4ckc0n 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Flag Semaphore Photo Decoding (DefCamp CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Two-Byte Nibble Reassembly with Random Padding (Trend Micro 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [block-mode-misuse-family.md](block-mode-misuse-family.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)
- [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)
- [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md)
- [exotic-secret-sharing-rabin-and-polynomials.md](exotic-secret-sharing-rabin-and-polynomials.md)

## 原始资料

- [classical-xor-and-substitution-ciphers.md](../raw/crypto/classical-xor-and-substitution-ciphers.md)
