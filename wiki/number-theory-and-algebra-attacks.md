---
type: technique
tags: [crypto, technique]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/number-theory-and-algebra-attacks.md
updated: 2026-05-22
---

# Number Theory and Algebra Attacks

## 适用场景

密钥恢复、原语误用、oracle、随机数、签名或协议假设是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 题目给出密文、nonce、签名、模数、oracle、PRNG 输出或自定义协议。
- 存在重复 nonce、弱随机、错误回显、数学结构或可查询接口。
- 源码能复现加密、签名、哈希或验证流程。
- 题面或 raw 线索能落到这些关键词之一：Elliptic Curve Isogenies、Pohlig-Hellman Attack (Weak ECC)、Baby-Step Giant-Step for General DLP、LLL Algorithm for Approximate GCD、Merkle-Hellman Knapsack Cryptosystem via LLL (ASIS 2014)、Coppersmith's Method (Close Private Keys)、Coppersmith's Method (Structured Primes, LACTF 2026)、Clock Group (x^2+y^2=1 mod p) DLP (LACTF 2026)。

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
| Elliptic Curve Isogenies | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Pohlig-Hellman Attack (Weak ECC) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Baby-Step Giant-Step for General DLP | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| LLL Algorithm for Approximate GCD | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Merkle-Hellman Knapsack Cryptosystem via LLL (ASIS 2014) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Coppersmith's Method (Close Private Keys) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Coppersmith's Method (Structured Primes, LACTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Clock Group (x^2+y^2=1 mod p) DLP (LACTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Quaternion RSA | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Polynomial Arithmetic in GF(2)[x] | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RSA Signing Bug | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Non-Permutation S-box Collision Attack (Nullcon 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Polynomial CRT in GF(2)[x] (Nullcon 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Manger's RSA Padding Oracle Attack (Nullcon 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| LWE Lattice Attack via CVP (EHAX 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Affine Cipher over Non-Prime Modulus (Nullcon 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Introspective CRC via GF(2) Linear Algebra (Google CTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Baby-Step Giant-Step for Sparse/Low Hamming Weight Exponents (SEC-T CTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Common Patterns | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

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
- [rsa-attacks.md](rsa-attacks.md)
- [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md)
- [zkp-secret-sharing-and-proof-systems.md](zkp-secret-sharing-and-proof-systems.md)
- [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md)
- [prng-z3-lcg-and-timing-attacks.md](prng-z3-lcg-and-timing-attacks.md)
- [lorenz-and-book-cipher-attacks.md](lorenz-and-book-cipher-attacks.md)

## 原始资料

- [number-theory-and-algebra-attacks.md](../raw/crypto/number-theory-and-algebra-attacks.md)
