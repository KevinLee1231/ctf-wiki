---
type: family
tags: [crypto, family, rsa, textbook-rsa]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/rsa-attacks.md
updated: 2026-06-12
---

# RSA Attacks

## 作用边界

本页是 RSA 基础路线 family，用于在看到 `n/e/c`、多组密文、多个模数、弱私钥指数、低指数、可查询 oracle 或可疑 prime 结构时，先判断该走哪条最短路线。它不再作为单一 technique，因为 raw 覆盖的是一组 RSA 攻击入口。

更长尾的素数结构、签名同态、fault、ROCA、CRT 泄露和复杂 oracle 转到 [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)。

## 首轮路由

| 信号 | 最小证据 | 下一跳 |
|---|---|---|
| `m^e < n`、`e=3/5/7`、无 padding | 直接整数开 e 次根能复算密文 | 本页 raw；必要时用 [crypto-tooling.md](crypto-tooling.md) |
| 同一 `n` 下多指数加密同一明文 | `gcd(e1,e2)=1`，有同模密文 | 本页 raw |
| 多个互素模数、同一明文、同一小 `e` | 密文数不少于 `e`，可 CRT 合并 | 本页 raw；线性 padding 转 [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| 小私钥指数 | `d < n^0.25` 或参数接近 Wiener/continued fraction 条件 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| 连续素数、近似素数、多素数 | `p≈q`、prime count 多、FactorDB/小因子有迹象 | 本页 raw；结构化 prime 转 specialized |
| OAEP/PKCS#1 padding oracle 或 timing oracle | 查询响应能区分区间、padding 或 timing | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| 题目变成多项式、GF(2)[x]、仿射模数 | RSA 只是外壳，核心是代数结构 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |

## 合并与拆分结论

- 保留为 family：它承接 RSA 基础攻击的首轮判断。
- 不与 `rsa-specialized-structures-and-oracles.md` 合并：基础路线强调快速排查，specialized 页强调复杂结构、oracle 和长尾案例。
- 不拆成小 e / 共模 / Wiener 等单页：当前这些分支作为 raw 速查足够，只有在 WP 案例积累出可迁移细节时再拆 technique。

## 常见误判

- 只看到 `e=65537` 就放弃 RSA；仍需检查 shared prime、prime structure、CRT 泄露和 oracle。
- 低指数攻击未先估 `m^e < n` 或 CRT 合并后的界。
- Manger/Bleichenbacher 类 oracle 没量化响应差异，导致查询脚本不可复现。
- 先跑重型 Sage/LLL，而没有做 FactorDB、gcd、近似平方根和参数规模首检。

## 关联页面

- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
- [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)
- [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)
- [crypto-tooling.md](crypto-tooling.md)

## 原始资料

- [rsa-attacks.md](../raw/crypto/rsa-attacks.md)
