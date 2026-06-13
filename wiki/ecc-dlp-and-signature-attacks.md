---
type: family
tags: [crypto, family, ecc, dlp, signature, nonce]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/ecc-dlp-and-signature-attacks.md
updated: 2026-06-12
---

# ECC DLP and Signature Attacks

## 适用场景

题目核心是曲线群、DLP、签名 nonce、small subgroup、invalid curve、异常曲线、同源/配对或 DSA/ECDSA/Schnorr 类签名关系。它是 crypto 首轮分流后的 ECC/签名 family 页，不是单一攻击技巧页。

## 识别信号

- 给出曲线参数、基点、点坐标、签名 `(r, s)`、公钥、oracle 或可控输入点。
- 曲线阶、子群阶、cofactor、判别式、j-invariant、异常阶或奇怪基点明显可疑。
- 多个签名存在相同 `r`、相近 nonce、低熵 nonce、hash collision 或 PRNG 生成的 nonce。
- 服务端没有检查点是否在曲线上、是否在正确子群中，或返回标量乘结果/验证结果差异。

## 最小证据

- 明确曲线定义、模数、基点阶和目标未知量：私钥、nonce、离散对数、共享密钥或明文。
- 至少一种可验证路线：群阶分解、签名方程、oracle 查询差异、点映射或 nonce 约束。
- 如果是签名问题，要保留消息哈希算法、签名格式和每个签名的 `(r, s, h)`。
- 如果是曲线结构问题，要先验证点合法性、阶、cofactor 和是否退化到更简单群。

## 解法骨架

1. 先做参数体检：曲线是否标准、阶是否 smooth、基点是否正确、点是否在曲线上。
2. DLP 路线先看群阶分解和可降维结构；签名路线先写出 nonce 方程。
3. 有 oracle 时先构造最小查询，确认是 small subgroup、invalid curve、torsion side channel 还是普通验证差异。
4. 得到候选私钥/nonce 后，用签名验证、标量乘或共享密钥派生做正向复算。

## 关键变体

| 变体 | 触发证据 | 处理路线 |
|---|---|---|
| smooth order DLP | 群阶可分解为小因子或 clock group 结构明显。 | Pohlig-Hellman / BSGS 分块求解，再 CRT 合并。 |
| invalid / singular / anomalous curve | 点不在曲线上、判别式为 0、阶等于 p 或曲线退化。 | 验证曲线结构，映射到乘法/加法群或使用 Smart attack。 |
| small subgroup / torsion | cofactor 大，服务端接受低阶点或 Ed25519 torsion 差异。 | 查询低阶点泄露私钥模小因子，收集后 CRT。 |
| repeated ECDSA/DSA nonce | 多签名 `r` 相同或 k 可复用。 | 由两个签名恢复 k 和私钥，再验证公钥。 |
| partial / biased nonce | nonce 高位/低位泄露、短 nonce、PRNG nonce 或 timing 泄露。 | 写 HNP/EHNP 关系，转 [lattice-and-lwe.md](lattice-and-lwe.md) 或 PRNG 页补状态。 |
| hash-collision generated k | nonce 派生依赖 MD5/SHA 碰撞或弱 hash。 | 先转 [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) 固定碰撞关系，再回签名方程。 |
| shared prime / modulus factor | 多个曲线或签名参数共享素因子。 | GCD 先做参数恢复，再进入 DLP/签名路线。 |

## 常见陷阱

- 只看曲线名，不检查实际参数、基点阶和 cofactor。
- 把 Schnorr、DSA、ECDSA 的签名方程混用，导致 nonce 恢复公式错误。
- 低阶点 oracle 没有收集足够模因子，CRT 结果不唯一。
- partial nonce 已进入格问题时，仍尝试暴力完整 nonce。

## 关联技巧

- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
- [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md)
- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)
- [crypto-tooling.md](crypto-tooling.md)

## 原始资料

- [ecc-dlp-and-signature-attacks.md](../raw/crypto/ecc-dlp-and-signature-attacks.md)
