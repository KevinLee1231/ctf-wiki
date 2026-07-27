---
type: technique
tags: [crypto, rsa, factorization, partial-key, structured-prime]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/rsa-attacks.md
  - ../raw/crypto/rsa-specialized-structures-and-oracles.md
  - ../raw/crypto/UMDCTF2024-key-recovery-wp.md
updated: 2026-07-27
---

# RSA Factor Relation and Partial-Key Recovery

## 适用场景

RSA 素因子并非独立随机，或私钥/CRT 参数存在共享因子、近似关系、已知位和截断字段，可把通用分解转成小范围搜索、GCD 或部分密钥恢复。

## 识别信号

- `p≈q`、连续素数、共享 prime、多素数，或 `p/q` 之间有和、差、异或、位反转关系。
- 给出 `dp/dq/qinv/d/phi` 的全部或部分位，或 PEM/DER 只损坏一段。
- 多个模数、hint 或幂表达式在模某一素因子后会消项。

## 最小证据

- 先完成 pairwise GCD、近似平方根、FactorDB 和字段边界检查。
- 将泄露关系写成模 `p`/模 `q` 或已知位约束，并量化未知量上界。
- 任一候选都必须满足 `p*q=n` 及全部 CRT/RSA 参数关系。

## 解法骨架

1. 从最低成本的 GCD、Fermat 和结构枚举开始。
2. 对 `edp-1=k(p-1)`、`ed-1=k phi(n)` 等关系枚举小乘子并取 GCD。
3. 已知高/低位时建立小根或比特传播模型，必要时再用格约简收尾。
4. 恢复因子后重建私钥并按原始 padding/编码解密。

## 关键变体

- 共享因子/近素数：GCD 或 Fermat 即可完成。
- 结构化 prime：利用生成器给出的位关系做逐位或低维搜索。
- 部分私钥：DER 字段、CRT 指数和 `ed` 同余可交叉传播未知位。

## 常见陷阱

- 一看到 RSA 就直接尝试通用整数分解。
- 忽略 `dp`、`dq` 与 `p-1`、`q-1` 的小乘子关系。
- 恢复出数值后未复核 DER 字段截断和消息预处理。

## 关联技巧

- [rsa-attacks.md](rsa-attacks.md)
- [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)
- [lattice-small-root-and-partial-leakage.md](lattice-small-root-and-partial-leakage.md)

## 原始资料

- [rsa-attacks.md](../raw/crypto/rsa-attacks.md)
- [rsa-specialized-structures-and-oracles.md](../raw/crypto/rsa-specialized-structures-and-oracles.md)
- [UMDCTF2024-key-recovery-wp](../raw/crypto/UMDCTF2024-key-recovery-wp.md)
