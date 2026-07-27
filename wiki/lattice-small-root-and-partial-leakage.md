---
type: technique
tags: [crypto, lattice, coppersmith, hnp, partial-leakage]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/lattice-and-lwe.md
  - ../raw/crypto/rsa-specialized-structures-and-oracles.md
updated: 2026-07-27
---

# Lattice Small-Root and Partial-Leakage Recovery

## 适用场景

秘密满足模多项式关系，且未知部分有可证明的小上界；或多组样本泄露 nonce、乘积、状态的高/低位，可构造 Coppersmith、HNP/CVP 或近似最短向量问题。

## 识别信号

- 已知 RSA prime/key 的连续高位或低位，剩余未知位较短。
- 多组 `a_i*x mod q` 只暴露高位/低位或带小误差。
- 关系可写为 `f(x)=0 mod N`，未知量个数少且 bounds 明确。

## 最小证据

- 写出精确多项式、模数和每个未知量的严格上界。
- 估算未知规模是否落在小根/HNP 可行区间。
- 说明格基每一行对应的代数关系与缩放，不只给脚本参数。

## 解法骨架

1. 先消去能用 GCD、线性代数或直接枚举处理的变量。
2. 按 monomial 和 modulus power 构造整系数格，并用 bounds 缩放列。
3. LLL/BKZ 后将短向量还原为整数多项式或近似关系。
4. 求公共根、验证原同余，再恢复完整密钥/状态。

## 关键变体

- Univariate Coppersmith：单个连续未知块。
- Multivariate small root：多个小变量，参数敏感且需要更强验证。
- HNP/CVP：多条带截断或噪声的线性模关系。

## 常见陷阱

- 用经验 bounds 但不区分数学上界和调参值。
- 格约简得到短向量后未验证其是否提供独立多项式。
- 问题本可直接 GCD/枚举，却先构造高维格。

## 关联技巧

- [lattice-and-lwe.md](lattice-and-lwe.md)
- [rsa-factor-relation-and-partial-key-recovery.md](rsa-factor-relation-and-partial-key-recovery.md)
- [linear-prng-state-and-seed-recovery.md](linear-prng-state-and-seed-recovery.md)
- [overcomplete-matrix-linear-relation-recovery.md](overcomplete-matrix-linear-relation-recovery.md)

## 原始资料

- [lattice-and-lwe.md](../raw/crypto/lattice-and-lwe.md)
- [rsa-specialized-structures-and-oracles.md](../raw/crypto/rsa-specialized-structures-and-oracles.md)
