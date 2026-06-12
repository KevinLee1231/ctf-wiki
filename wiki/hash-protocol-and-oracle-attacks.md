---
type: technique
tags: [crypto, technique]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/hash-protocol-and-oracle-attacks.md
updated: 2026-05-21
---

# Hash, Protocol and Oracle Attacks

## 适用场景

密钥恢复、原语误用、oracle、随机数、签名或协议假设是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 题目给出密文、nonce、签名、模数、oracle、PRNG 输出或自定义协议。
- 存在重复 nonce、弱随机、错误回显、数学结构或可查询接口。
- 源码能复现加密、签名、哈希或验证流程。
- 题面或 raw 线索能落到这些关键词之一：Blum-Goldwasser Bit-Extension Oracle (PlaidCTF 2013)、Hash Length Extension Attack (PlaidCTF 2014)、Hash Length Extension Attack、Compression Oracle / CRIME-Style Attack (BCTF 2015)、Compression Oracle (CRIME-Style)、Hash Function Time Reversal via Cycle Detection (BSidesSF 2025)、OFB Mode with Invertible RNG Backward Decryption (BSidesSF 2026)、Weak Key Derivation via Public Key Hash XOR (BSidesSF 2026)。

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
| Blum-Goldwasser Bit-Extension Oracle (PlaidCTF 2013) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Hash Length Extension Attack (PlaidCTF 2014) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Hash Length Extension Attack | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Compression Oracle / CRIME-Style Attack (BCTF 2015) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Compression Oracle (CRIME-Style) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Hash Function Time Reversal via Cycle Detection (BSidesSF 2025) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| OFB Mode with Invertible RNG Backward Decryption (BSidesSF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Weak Key Derivation via Public Key Hash XOR (BSidesSF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| HMAC-CRC Linearity Attack (Boston Key Party 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| DES Weak Keys in OFB Mode (Boston Key Party 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Square Attack on Reduced-Round AES (0CTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| SRP (Secure Remote Password) Protocol Bypass via Modular Arithmetic (ASIS CTF Finals 201… | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Modified AES S-Box Brute-Force Recovery (H4ckIT CTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| AES-ECB Byte-at-a-Time Chosen Plaintext (ABCTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| AES-ECB Cut-and-Paste Block Manipulation (NDH Quals 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| AES-CBC IV Bit-Flip Authentication Bypass (Google CTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Rabin Cryptosystem LSB Parity Oracle (PlaidCTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| PBKDF2 Pre-Hash Bypass for Long Passwords (BackdoorCTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| MD5 Multi-Collision (BackdoorCTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Custom Hash State Reversal via Known Intermediates (BackdoorCTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| CRC32 Brute-Force for Small Payloads (BackdoorCTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Noisy RSA LSB Oracle with Post-Hoc Error Correction (SharifCTF 7 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Sponge Hash Collision via Meet-in-the-Middle on Partial State (BKP 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| CBC IV Forgery + Block Truncation for Authentication Bypass (0CTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [block-mode-misuse-family.md](block-mode-misuse-family.md)
- [classical-xor-and-substitution-ciphers.md](classical-xor-and-substitution-ciphers.md)
- [compare-breakpoint-plaintext-recovery.md](compare-breakpoint-plaintext-recovery.md)
- [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)
- [embedded-python-pyd-custom-aes.md](embedded-python-pyd-custom-aes.md)

## 原始资料

- [hash-protocol-and-oracle-attacks.md](../raw/crypto/hash-protocol-and-oracle-attacks.md)
