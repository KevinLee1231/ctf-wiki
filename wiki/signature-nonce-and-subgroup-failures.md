---
type: technique
tags: [crypto, signature, nonce-reuse, subgroup, ecc]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/ecc-dlp-and-signature-attacks.md
  - ../raw/crypto/zkp-secret-sharing-and-proof-systems.md
updated: 2026-07-27
---

# Signature Nonce and Subgroup Failures

## 适用场景

ECDSA/DSA/Schnorr 类签名复用、偏置或部分泄露 nonce，或协议未验证群元素/子群归属，使私钥可由代数消元、HNP 或小子群查询恢复。

## 识别信号

- 多个签名出现相同 `r`，nonce 来自弱 PRNG，或已知部分 nonce 位。
- 服务接受攻击者提供的点/群元素但不检查曲线、阶或 cofactor。
- 协议 transcript 对消息、身份或 challenge 的绑定不完整。

## 最小证据

- 精确确认签名方程、消息 hash 编码和模群阶运算。
- 复用攻击需证明 nonce 相同；部分泄露需给出位数与样本量。
- 子群攻击需确认服务确实对未验证元素执行秘密标量运算。

## 解法骨架

1. 统一所有签名为模 `q` 线性方程。
2. 相同 nonce 直接联立消元；部分 nonce 建 HNP 格并验证候选私钥。
3. 无效曲线/小子群时选择已知小阶点，按响应恢复标量模各小因子。
4. 用 CRT 合并泄露并以公钥或新签名验证。

## 关键变体

- Exact nonce reuse：两个签名即可恢复 nonce 和私钥。
- Biased/partial nonce：需要多样本和正确 bounds。
- Invalid-curve/small-subgroup：依赖缺失点验证和可观察响应。

## 常见陷阱

- 把 `r` 相同但消息 hash 编码不同的样本直接代公式。
- 使用模 `p` 而非模群阶 `q` 计算签名方程。
- 忽略 cofactor clearing 或库已执行的点验证。

## 关联技巧

- [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)
- [lattice-small-root-and-partial-leakage.md](lattice-small-root-and-partial-leakage.md)
- [zkp-secret-sharing-and-proof-systems.md](zkp-secret-sharing-and-proof-systems.md)

## 原始资料

- [ecc-dlp-and-signature-attacks.md](../raw/crypto/ecc-dlp-and-signature-attacks.md)
- [zkp-secret-sharing-and-proof-systems.md](../raw/crypto/zkp-secret-sharing-and-proof-systems.md)
