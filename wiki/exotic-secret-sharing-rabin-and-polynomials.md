---
type: family
tags: [crypto, family, secret-sharing, rabin, polynomial, mnemonic]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/exotic-secret-sharing-rabin-and-polynomials.md
  - ../raw/crypto/D3CTF2022-d3share-wp.md
  - ../raw/crypto/VNCTF2026-mymnemonic-wp.md
updated: 2026-07-06
---

# Exotic Secret Sharing, Rabin and Polynomials

## 作用边界

本页是 crypto 长尾结构 family，覆盖 Cayley-Purser、BIP39 部分助记词、Asmuth-Bloom/CRT secret sharing、Rabin 四根解密、多项式素数、LCG 周期检测和 Vandermonde 系数恢复。共同点是题面参数不落入常规 RSA/ECC/AES，但可通过结构识别降维。

## 识别信号

- 出现 shares、threshold、CRT、mnemonic、Rabin、四个平方根、多项式系数、Vandermonde、周期检测或少见公钥原语。
- 常规 RSA/ECC/分组模式判断不匹配，但源码能写出清晰的代数关系。
- 成功条件通常是恢复 secret、私钥等价物、缺失助记词、消息根或多项式系数。

## 最小证据

- 原语定义、参数规模、模数/域、已知样本数量和未知量边界。
- 能写出方程或校验过程：CRT 重构、Rabin roots、checksum、线性系统、周期状态。
- 对 BIP39/助记词，确认词表、checksum、缺失位置和派生路径。
- 对 secret sharing，确认阈值方案，不要把所有 shares 都当 Shamir。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| BIP39 熵/部分缺词 | 词表、ENT/CS 边界、checksum、缺失位和派生地址/密钥 | 补 checksum 或枚举缺词后验证 seed/地址 |
| CRT threshold sharing | shares 与模数是否满足 Asmuth-Bloom 条件 | CRT 合并后检查 secret 范围 |
| 扰动多项式 key sharing | node 私钥是否为主多项式插值项加 `g/h` 小扰动，且已有多个身份点私钥 | 先做拉格朗日插值，再按小值结构转 [lattice-and-lwe.md](lattice-and-lwe.md) 或子集和分支 |
| Rabin 四根 | `c = m^2 mod n`，n 可分解或给定 p/q | 四根 CRT 组合并用格式筛选 |
| polynomial primes | p/q 由多项式或系数生成 | 先恢复系数或构造因式关系 |
| Vandermonde 系数 | 多点函数值和次数界 | 线性系统恢复多项式 |
| LCG period | 输出无限或周期明显 | 先转 [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| Cayley-Purser/低频原语 | 私钥不可得但群/矩阵结构可逆 | 写出等价解密关系 |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [exotic-secret-sharing-rabin-and-polynomials.md](../raw/crypto/exotic-secret-sharing-rabin-and-polynomials.md) | 原始资料内覆盖 Cayley-Purser 等价解密、Asmuth-Bloom CRT sharing、Rabin 四根/多项式素数、Vandermonde 系数恢复和 LCG 周期检测；这些都先写结构方程，再决定转 CRT、线性系统、Rabin roots 或 PRNG。 |
| [D3CTF2022-d3share-wp](../raw/crypto/D3CTF2022-d3share-wp.md) | 扰动多项式 secret sharing 不能当普通 Shamir；先用拉格朗日插值得到扰动组合，再利用小值结构、LLL/子集和或三点等式分辨 `g/h` 分支。 |
| [VNCTF2026-mymnemonic-wp](../raw/crypto/VNCTF2026-mymnemonic-wp.md) | 192-bit 数据不是完整助记词索引流而是 BIP39 `ENT`；先按 `CS = ENT / 32` 补 SHA256 checksum，再按 11-bit 映射中文词表并 PBKDF2 派生 seed。 |

## 资料内长尾模式

| 模式 | 最小识别信号 | 先做什么 |
|---|---|---|
| Cayley-Purser | 公钥矩阵/群元素和 ciphertext 关系可被重排，私钥缺失但存在等价缩放或共轭结构。 | 先写出公开量之间的可逆关系，找等价解密因子，不要按常规 RSA/ECC 处理。 |
| Asmuth-Bloom | shares 搭配两两互素模数，重构依赖 CRT 和 secret 范围约束。 | 先确认模数大小条件和阈值，再 CRT 合并候选 secret。 |
| Rabin polynomial primes | `c = m^2 mod n` 且 p/q 由多项式或可恢复参数生成。 | 先恢复/分解 p、q，再 CRT 组合四根并用格式筛选。 |
| Vandermonde recovery | 多个点值和次数界足以恢复多项式系数。 | 按域/模数建立线性系统，解系数后 forward check。 |
| LCG period detection | 输出足够长且可观察重复周期，但缺少传统 `a,c,m` 参数。 | 先找 period_start/period，再预测后续输出；若要恢复参数再转 PRNG family。 |

## 合并与拆分结论

- 保留为 family：这些长尾原语共享“先识别结构再降维”的价值，但每个单独案例不足以单独成页。
- 不合并进 `number-theory-and-algebra-attacks.md`：该页覆盖通用数论/代数主线，本页保留少见协议和资料入口。
- 不合并进 `mt-lcg-and-seed-recovery.md`：LCG period 只作为交叉变体，主线仍是长尾结构。

## 常见误判

- 把 Asmuth-Bloom 当 Shamir 插值，方向完全错误。
- Rabin 解出四根后不做格式/冗余校验。
- BIP39 只爆词，不验证 checksum 和派生路径。
- 长尾原语没先写方程，直接套常规 RSA/ECC 工具。

## 关联页面

- [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md)
- [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)
- [crypto-tooling.md](crypto-tooling.md)

## 原始资料

- [exotic-secret-sharing-rabin-and-polynomials.md](../raw/crypto/exotic-secret-sharing-rabin-and-polynomials.md)
- [D3CTF2022-d3share-wp](../raw/crypto/D3CTF2022-d3share-wp.md)
- [VNCTF2026-mymnemonic-wp](../raw/crypto/VNCTF2026-mymnemonic-wp.md)
