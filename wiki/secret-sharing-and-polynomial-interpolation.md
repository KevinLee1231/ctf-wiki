---
type: technique
tags: [crypto, secret-sharing, shamir, interpolation, polynomial]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/exotic-secret-sharing-rabin-and-polynomials.md
  - ../raw/crypto/zkp-secret-sharing-and-proof-systems.md
  - ../raw/crypto/UMDCTF2023-adi-shamirs-sharing-system-wp.md
updated: 2026-07-28
---

# Secret Sharing and Polynomial Interpolation

## 适用场景

秘密被编码为 Shamir/Asmuth-Bloom/自定义多项式份额，或多个点值、承诺和缺失 share 足以通过插值、CRT 或约束关系恢复常数项。

## 识别信号

- 每份数据包含 `(x,y)`、阈值 `t`、模素数或多项式承诺。
- 份额数达到阈值，或少量份额错误/缺失但有额外校验。
- 题目给出 Vandermonde 型方程、BIP39/助记词 share 或 CRT residue。

## 最小证据

- 确认运算域、阈值、x 坐标唯一性和秘密所在系数。
- 检查 share 是否经过编码、偏移、哈希或模数映射。
- 对错误份额场景估计可纠正数量并保留一致性校验。

## 解法骨架

1. 解析份额并映射到正确有限域/整数 CRT 系统。
2. 用 Lagrange 插值恢复目标系数；缺失份额可直接插值，错误份额用一致子集或纠错模型。
3. 若有承诺/KZG，先验证份额，避免污染插值。
4. 将恢复整数按题目字节序和长度解码，再用格式或承诺复验。

## 关键变体

- Shamir：有限域多项式常数项是秘密。
- CRT/Asmuth-Bloom：份额是不同模数下的 residue。
- Polynomial proof：份额还需满足 pairing/commitment 验证。

## 常见陷阱

- 在有理数或整数域而非 `GF(p)` 插值。
- x 坐标重复或 share 排序被误当作 x。
- 恢复值丢失前导零，导致字节串错位。

## 关联技巧

- [exotic-secret-sharing-rabin-and-polynomials.md](exotic-secret-sharing-rabin-and-polynomials.md)
- [zkp-secret-sharing-and-proof-systems.md](zkp-secret-sharing-and-proof-systems.md)
- [algebraic-polynomial-and-modular-root-reconstruction.md](algebraic-polynomial-and-modular-root-reconstruction.md)

## 原始资料

- [exotic-secret-sharing-rabin-and-polynomials.md](../raw/crypto/exotic-secret-sharing-rabin-and-polynomials.md)
- [zkp-secret-sharing-and-proof-systems.md](../raw/crypto/zkp-secret-sharing-and-proof-systems.md)
- [UMDCTF2023-adi-shamirs-sharing-system-wp](../raw/crypto/UMDCTF2023-adi-shamirs-sharing-system-wp.md)
