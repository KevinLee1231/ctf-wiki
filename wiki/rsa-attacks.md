---
type: technique
tags: [crypto, technique]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/rsa-attacks.md
updated: 2026-05-21
---

# RSA Attacks

## 适用场景

密钥恢复、原语误用、oracle、随机数、签名或协议假设是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 题目给出密文、nonce、签名、模数、oracle、PRNG 输出或自定义协议。
- 存在重复 nonce、弱随机、错误回显、数学结构或可查询接口。
- 源码能复现加密、签名、哈希或验证流程。
- 题面或 raw 线索能落到这些关键词之一：Small Public Exponent (Cube Root)、Common Modulus Attack、Wiener's Attack (Small Private Exponent)、Pollard's p-1 Factorization、Hastad's Broadcast Attack、Hastad Broadcast Attack with Linear Padding -- Coppersmith (PlaidCTF 2017)、Franklin-Reiter Related Message Attack on RSA e=3 (N1CTF 2018)、Coppersmith Attack on Linearly-Related RSA Primes (ASIS CTF 2018)。

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
| Small Public Exponent (Cube Root) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Common Modulus Attack | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Wiener's Attack (Small Private Exponent) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Pollard's p-1 Factorization | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Hastad's Broadcast Attack | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Hastad Broadcast Attack with Linear Padding -- Coppersmith (PlaidCTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Franklin-Reiter Related Message Attack on RSA e=3 (N1CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Coppersmith Attack on Linearly-Related RSA Primes (ASIS CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RSA with Consecutive Primes (Fermat Factorization) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Multi-Prime RSA | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RSA with Restricted-Digit Primes (LACTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Coppersmith for Structured RSA Primes (LACTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Manger's RSA Padding Oracle Attack (Nullcon 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Manger's Attack on RSA-OAEP via Timing Oracle (HTB Early Bird) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Polynomial Hash with Trivial Root (Pragyan 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Polynomial CRT in GF(2)[x] (Nullcon 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Affine Cipher over Non-Prime Modulus (Nullcon 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RSA Attacks | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

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

- [rsa-attacks.md](../raw/crypto/rsa-attacks.md)
