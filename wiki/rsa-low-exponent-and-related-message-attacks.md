---
type: technique
tags: [crypto, rsa, low-exponent, common-modulus, related-message]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/rsa-attacks.md
  - ../raw/crypto/UMDCTF2023-eeveelutions-wp.md
updated: 2026-07-27
---

# RSA Low-Exponent and Related-Message Attacks

## 适用场景

RSA 使用小公开指数、无随机 padding，或同一明文在同模数/多模数下以相关形式重复加密，使密文关系可在整数域或低次多项式中消去。

## 识别信号

- `e=3/5/7`，消息短、前缀已知或明显没有 OAEP/PKCS#1 随机化。
- 同一明文对应多个互素模数，或同一模数对应互素公开指数。
- 两条明文只差短后缀、线性偏移或可参数化的小差值。

## 最小证据

- 验证 `m^e < n`，或广播 CRT 合并后的值仍等于整数 `m^e`。
- 共模攻击需有同一 `n`、同一明文及 `gcd(e1,e2)=1`。
- 相关消息攻击需写出低次关系并确认小差值界。

## 解法骨架

1. 统一参数，先排查直接整数开根和共模 Bezout 组合。
2. 多模数同消息时 CRT 合并密文，再做精确整数开根。
3. 相关消息时构造 Franklin-Reiter 多项式；短未知差值先用小根恢复。
4. 将候选重新加密，严格验证全部密文关系和消息编码。

## 关键变体

- 裸 RSA 小消息：无需分解模数，直接整数开根。
- Hastad broadcast：要求足够多互素模数且明文一致。
- Short-pad/Franklin-Reiter：先恢复消息差，再消去得到原文。

## 常见陷阱

- 未证明整数开根条件，只因 `e` 小就直接取根。
- 不同实例的明文 padding 实际不相同。
- 使用浮点开根导致大整数边界误判。

## 关联技巧

- [rsa-attacks.md](rsa-attacks.md)
- [rsa-factor-relation-and-partial-key-recovery.md](rsa-factor-relation-and-partial-key-recovery.md)
- [lattice-small-root-and-partial-leakage.md](lattice-small-root-and-partial-leakage.md)

## 原始资料

- [rsa-attacks.md](../raw/crypto/rsa-attacks.md)
- [UMDCTF2023-eeveelutions-wp](../raw/crypto/UMDCTF2023-eeveelutions-wp.md)
