---
type: technique
tags: [crypto, technique]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/rsa-specialized-structures-and-oracles.md
updated: 2026-05-21
---

# RSA Specialized Structures and Oracles

## 适用场景

密钥恢复、原语误用、oracle、随机数、签名或协议假设是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 题目给出密文、nonce、签名、模数、oracle、PRNG 输出或自定义协议。
- 存在重复 nonce、弱随机、错误回显、数学结构或可查询接口。
- 源码能复现加密、签名、哈希或验证流程。
- 题面或 raw 线索能落到这些关键词之一：RSA p=q Validation Bypass (BearCatCTF 2026)、RSA Cube Root CRT when gcd(e, phi) > 1 (BearCatCTF 2026)、Factoring n from Multiple of phi(n) (BearCatCTF 2026)、RSA Signature Forgery via Multiplicative Homomorphism (MMA CTF 2015)、RSA Multiplicative Homomorphism Signature Forgery、Weak RSA Key Generation via Base Representation (Sharif CTF 2016)、RSA with gcd(e, phi(n)) > 1 (CSAW 2015)、Batch GCD for Shared Prime Factoring (BSidesSF 2025)。

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
| RSA p=q Validation Bypass (BearCatCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RSA Cube Root CRT when gcd(e, phi) > 1 (BearCatCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Factoring n from Multiple of phi(n) (BearCatCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RSA Signature Forgery via Multiplicative Homomorphism (MMA CTF 2015) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RSA Multiplicative Homomorphism Signature Forgery | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Weak RSA Key Generation via Base Representation (Sharif CTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RSA with gcd(e, phi(n)) > 1 (CSAW 2015) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Batch GCD for Shared Prime Factoring (BSidesSF 2025) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RSA Partial Key Recovery from dp dq qinv (0CTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RSA-CRT Fault Attack / Bit-Flip Recovery (CSAW CTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RSA Homomorphic Decryption Oracle Bypass (ECTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RSA with Small Prime Factors and CRT Decomposition (Hack The Vote 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RSA Timing Attack on Montgomery Reduction (DEF CON 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Bleichenbacher Low-Exponent RSA Signature Forgery (Google CTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Coppersmith Small Roots for Linearly Related Primes (Tokyo Westerns 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| ROCA Attack on RSA CVE-2017-15361 (EasyCTF IV) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RSA Signature Bypass with e=1 and Crafted Modulus (BackdoorCTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Dependent-Prime RSA: q = e^-1 mod p (TokyoWesterns CTF 4th 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RSA Three-Key Pairwise GCD Triangle (Trend Micro 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RSA n = p^2q Schmidt-Samoa Variant (ASIS Finals 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Modulus Recovery via GCD of Encryption Residuals (X-MAS CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Textbook RSA Negation via encrypt(-1) (X-MAS CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Poly-Exponent RSA: GCD of p^p Combinations (ASIS Finals 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Biased LSB Oracle with Mode-of-Runs Recovery (CSAW CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

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

- [rsa-specialized-structures-and-oracles.md](../raw/crypto/rsa-specialized-structures-and-oracles.md)
