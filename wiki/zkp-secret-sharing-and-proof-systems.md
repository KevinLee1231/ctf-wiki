---
type: technique
tags: [crypto, technique]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/zkp-secret-sharing-and-proof-systems.md
updated: 2026-05-21
---

# ZKP, Secret Sharing and Proof Systems

## 适用场景

密钥恢复、原语误用、oracle、随机数、签名或协议假设是主要障碍。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 题目给出密文、nonce、签名、模数、oracle、PRNG 输出或自定义协议。
- 存在重复 nonce、弱随机、错误回显、数学结构或可查询接口。
- 源码能复现加密、签名、哈希或验证流程。
- 题面或 raw 线索能落到这些关键词之一：ZKP Attacks、Graph 3-Coloring、Z3 SMT Solver Guide、Garbled Circuits: Free XOR Delta Recovery (LACTF 2026)、Bigram/Trigram Substitution -> Constraint Solving (LACTF 2026)、Shamir Secret Sharing with Deterministic Coefficients (LACTF 2026)、Race Condition in Crypto-Protected Endpoints (LACTF 2026)、Garbled Circuits: AES Key Recovery via Metadata Leakage (srdnlenCTF 2026)。

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
| ZKP Attacks | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Graph 3-Coloring | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Z3 SMT Solver Guide | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Garbled Circuits: Free XOR Delta Recovery (LACTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Bigram/Trigram Substitution -> Constraint Solving (LACTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Shamir Secret Sharing with Deterministic Coefficients (LACTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Race Condition in Crypto-Protected Endpoints (LACTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Garbled Circuits: AES Key Recovery via Metadata Leakage (srdnlenCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Post-Quantum Signature Fault Injection: MAYO (srdnlenCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Lattice-Based Threshold Signature Attack: FROST (srdnlenCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Groth16 Broken Trusted Setup — delta == gamma (DiceCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Groth16 Proof Replay — Unconstrained Nullifier (DiceCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| DV-SNARG Forgery via Verifier Oracle (DiceCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| KZG Pairing Oracle for Permutation Recovery (UNbreakable 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Shamir Secret Sharing with Reused Polynomial Coefficients (PoliCTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| ZKP & Constraint Solving | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

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

- [zkp-secret-sharing-and-proof-systems.md](../raw/crypto/zkp-secret-sharing-and-proof-systems.md)
