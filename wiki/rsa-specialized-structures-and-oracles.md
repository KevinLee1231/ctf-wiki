---
type: family
tags: [crypto, family, rsa, coppersmith, oracle, signature]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/rsa-specialized-structures-and-oracles.md
updated: 2026-06-12
---

# RSA Specialized Structures and Oracles

## 作用边界

本页是 RSA 长尾结构和 oracle family。它承接那些已经超出基础 RSA 排查的题：特殊素数关系、`gcd(e, phi)` 非 1、CRT 参数泄露、同态签名、fault、ROCA、padding oracle、LSB oracle、可构造模数、长度/截断 bug 和 Coppersmith 小根模型。

基础小 e、共模、Hastad、Wiener、Fermat、多素数等快速排查先看 [rsa-attacks.md](rsa-attacks.md)。

## 变体路由

| 信号 | 先确认 | 下一跳 |
|---|---|---|
| prime 有线性关系、已知高/低/中间位、base representation | 未知量界是否满足 Coppersmith，小根变量是一元还是多元 | [lattice-and-lwe.md](lattice-and-lwe.md)、[number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| `gcd(e, phi(n)) > 1`、多重根、prime power | 每个模 prime 下有多少根，是否能 CRT 枚举并用格式筛选 | 本页 raw |
| 泄露 `dp/dq/qinv`、CRT fault、bit flip | CRT 参数能否反推出 p/q，或故障签名和正确签名的 gcd 是否暴露因子 | 本页 raw |
| 签名同态、message factoring、`e=1`、可构造 modulus | 验证逻辑是否只检查 textbook RSA 关系或 PKCS 前缀 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) |
| decryption oracle、LSB oracle、Manger/Bleichenbacher | 响应差异是否可稳定压缩明文区间 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) |
| Batch GCD、shared prime、ROCA | 是否有多公钥集合、模数指纹或可批量 gcd 的输入 | [crypto-tooling.md](crypto-tooling.md) |
| strlen/NULL/last-byte overwrite、协议包装 bug | RSA 漏洞是否来自实现层截断或解析差异 | [json-duplicate-key-hmac-parser-differential.md](json-duplicate-key-hmac-parser-differential.md) |

## 合并与拆分结论

- 保留为 family：raw 覆盖大量互不相同的 RSA 长尾结构，页面价值是判断建模路线。
- 不与 `rsa-attacks.md` 合并：基础页用于快速排查，specialized 页用于需要建模、oracle 或实现 bug 的题。
- 不拆每种长尾攻击：当前 raw 多为短案例速查；后续若 Coppersmith/RSA oracle 有多篇 WP 支撑，可拆具体 technique。

## 常见误判

- 把所有 RSA 都先丢给通用工具，忽略题目给的结构关系。
- Coppersmith 未先估未知量界，导致格维度和 bound 乱试。
- Oracle 只看一次响应，没有证明响应差异和明文区间存在稳定关系。
- 签名题只关注 hash 算法，忽略 textbook RSA 乘法同态和验签前缀检查。

## 关联页面

- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
- [rsa-attacks.md](rsa-attacks.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)
- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)
- [crypto-tooling.md](crypto-tooling.md)

## 原始资料

- [rsa-specialized-structures-and-oracles.md](../raw/crypto/rsa-specialized-structures-and-oracles.md)
