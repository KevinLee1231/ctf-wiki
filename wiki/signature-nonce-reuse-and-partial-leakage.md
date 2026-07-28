---
type: technique
tags: [crypto, signature, nonce-reuse, partial-nonce, hnp, ecc]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/ecc-dlp-and-signature-attacks.md
  - ../raw/crypto/zkp-secret-sharing-and-proof-systems.md
  - ../raw/crypto/D3CTF2019-sign2win-wp.md
  - ../raw/crypto/D3CTF2022-leak-dsa-wp.md
  - ../raw/crypto/VNCTF2026-schnorr-wp.md
updated: 2026-07-28
---

# Signature Nonce Reuse and Partial Leakage

## 适用场景

ECDSA、DSA、Schnorr 或同类线性签名/证明协议复用 nonce，nonce 可预测、带偏置或泄露部分位；目标是通过直接消元、special soundness 或 Hidden Number Problem 恢复 nonce、witness 或长期私钥。

## 识别信号

- 不同消息的签名出现相同 `r`，或 Schnorr transcript 重复 commitment。
- nonce 由固定 seed、时间、弱 PRNG 或可控输入生成。
- 每个样本泄露 nonce 的高位、低位、短误差或线性关系。
- 服务返回多个可比较 transcript，且 challenge/message hash 的绑定可精确复现。

## 最小证据

- 写出实际实现的签名/响应方程、消息编码和模群阶 `q` 运算。
- exact reuse 必须证明 nonce/commitment 相同；partial leakage 必须量化泄露位数、误差界和样本量。
- 私钥候选能重建公钥，并验证至少一个未参与求解的签名或 transcript。

## 解法骨架

1. 把每个样本统一为模 `q` 的线性方程，先核对 hash-to-integer 和签名编码。
2. 相同 nonce/commitment 时联立两式消去随机数；Schnorr 可由两个不同 challenge 直接提取 witness。
3. 只有部分 nonce 信息时，构造 HNP 格并按泄露方向设置缩放与 bounds。
4. 对 LLL/BKZ 候选恢复 nonce 和私钥，使用公钥及新签名正向验证。

## 关键变体

| 变体 | 决定性证据 | 处理 |
|---|---|---|
| ECDSA/DSA exact reuse | 相同 `r` 且消息摘要不同 | 两个方程直接消元。 |
| Schnorr commitment reuse | 相同承诺对应不同 challenge | 响应方程相减提取 witness。 |
| biased/partial nonce | 多样本共享已知位或小误差 | 建 HNP 格，不能套 exact-reuse 公式。 |
| 构造式复用 | 可选择公钥、私钥关系或消息 | 从验证等式反向构造共同签名条件。 |

## 常见陷阱

- 使用模曲线素数 `p` 而不是模群阶 `q`。
- 只凭 `r` 相同就套公式，却没有复现消息摘要到整数的编码。
- HNP 的已知位方向、中心化或缩放写反，LLL 仍返回“看似很短”的错误向量。
- 把点验证缺失混进 nonce 页面；那类问题应转无效曲线/小子群技巧。

## 关联技巧

- [invalid-curve-and-small-subgroup-attacks.md](invalid-curve-and-small-subgroup-attacks.md)
- [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)
- [lattice-small-root-and-partial-leakage.md](lattice-small-root-and-partial-leakage.md)
- [zkp-secret-sharing-and-proof-systems.md](zkp-secret-sharing-and-proof-systems.md)

## 原始资料

- [ecc-dlp-and-signature-attacks.md](../raw/crypto/ecc-dlp-and-signature-attacks.md)
- [zkp-secret-sharing-and-proof-systems.md](../raw/crypto/zkp-secret-sharing-and-proof-systems.md)
- [D3CTF2019-sign2win-wp](../raw/crypto/D3CTF2019-sign2win-wp.md)
- [D3CTF2022-leak-dsa-wp](../raw/crypto/D3CTF2022-leak-dsa-wp.md)
- [VNCTF2026-schnorr-wp](../raw/crypto/VNCTF2026-schnorr-wp.md)
