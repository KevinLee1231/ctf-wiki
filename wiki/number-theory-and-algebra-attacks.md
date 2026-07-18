---
type: family
tags: [crypto, family, number-theory, algebra, dlp, coppersmith, polynomial]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/number-theory-and-algebra-attacks.md
  - ../raw/crypto/WMCTF2025-lemonpepper-wp.md
  - ../raw/crypto/0xGame2020-week1-Fibonacci-wp.md
updated: 2026-07-18
---

# Number Theory and Algebra Attacks

## 作用边界

本页是数论与代数攻击 family，覆盖 DLP/BSGS/Pohlig-Hellman、ECC/isogeny、approximate GCD、Merkle-Hellman、Coppersmith、特殊群、四元数 RSA、GF(2)[x] 多项式、p-adic/Hensel、CRC 线性代数和 Manger oracle 等路线。

它的职责是判断数学结构属于哪一类，而不是把所有代数题合并成一个通用求解步骤。若目标已经明确是 RSA 常见参数、LWE 格问题、ECC 签名 nonce 或随机数恢复，应优先转到更窄页面。

## 识别信号

- 源码中出现群运算、模多项式、曲线阶、smooth order、小根、oracle 区间、GF(2) 线性系统、特殊矩阵或非标准代数结构。
- 题目给出足够多的样本或可查询接口，用于构造方程、CRT、gcd、DLP、root lifting 或 oracle 搜索。
- 普通 RSA/ECC/PRNG 套路解释不完，需要识别实现者自定义的代数对象。

## 最小证据

- 写清代数对象：整数模环、有限域、多项式环、曲线群、矩阵群、四元数、clock group 或 p-adic 环。
- 明确攻击目标：求离散对数、分解模数、求小根、恢复状态、伪造签名、解线性系统或缩小 oracle 区间。
- 先在小参数上复现群律、CRT、root lifting 或 oracle 收敛。
- 解出结果必须回代到协议状态、签名、密文或校验函数。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| ECC order smooth、低阶子群、DLP、isogeny | 先分解阶并判断 PH/BSGS/特殊曲线，而不是直接爆破私钥 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| Coppersmith、小根、结构化素数、close private key | 先写多项式和根界，再决定 RSA 页或格页 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md), [lattice-and-lwe.md](lattice-and-lwe.md) |
| approximate GCD、knapsack、subset sum | 先判断是否短向量/最近向量问题，再转格 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| 递推序列模小整数、Pisano period、周期求和 | 先恢复周期和求和语义，再看结果是否喂给 RSA、PRNG 或其它协议 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| 四元数 RSA、clock group、矩阵 ElGamal、GF(2)[x] | 先恢复群律/环结构，再找可线性化或 CRT 的不变量 | [homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md) |
| p-adic 或 Hensel lift 卡在重根 | 先看导数和重根结构，必要时分支枚举候选再结合协议状态剪枝 | [crypto-tooling.md](crypto-tooling.md) |
| Manger/padding oracle、区间 oracle | 先证明 oracle 单调或区间收缩，再写可重放查询器 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md), [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) |
| CRC、affine cipher 非素数模、GF(2) 线性系统 | 先确定域/环和矩阵方向，再做线性代数 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md), [classical-xor-and-substitution-ciphers.md](classical-xor-and-substitution-ciphers.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-lemonpepper-wp](../raw/crypto/WMCTF2025-lemonpepper-wp.md) | `Lemon` 部分是模 `q^e` 上的重根多项式，普通 `.roots()`/Hensel 可能卡住；先通过求导保留高重数根，再结合 `Pepper` 的 p-adic 候选和 MCG 递推剪枝恢复状态。 |
| [0xGame2020-week1-Fibonacci-wp](../raw/crypto/0xGame2020-week1-Fibonacci-wp.md) | Fibonacci 模 `n` 的长和可用 Pisano period 压缩到一个周期内计算；求和下标要与题目代码一致，结果再回到 RSA 素数生成结构。 |
| [ACTF2026-inverse-pow-wp](../raw/crypto/ACTF2026-inverse-pow-wp.md) | `2^n` 十进制前缀匹配可转成 `{n log10(2)}` 命中窄区间，先用固定点 BSGS/LLL 找足够小的 `n`。 |
| [ACTF2026-pandora-wp](../raw/crypto/ACTF2026-pandora-wp.md) | 二次域 ideal / binary quadratic form 泄露判别式小数、`A` 高位和 `B`，先恢复判别式与 `A`，再做二元 Coppersmith 小根。 |
| [D3CTF2019-noise-wp](../raw/crypto/D3CTF2019-noise-wp.md) | 带噪声取模 oracle 需要维护候选区间，构造固定商查询把随机噪声转成强收缩。 |
| [HGAME2026-eezzdlp-wp](../raw/crypto/HGAME2026-eezzdlp-wp.md) | 矩阵 DLP 在 `n=p^2` 且 `det(a)=1` 时不能走行列式；改用特征值上的 p-adic log 和 Pohlig-Hellman 补低位。 |
| [HGAME2026-ezdlp-wp](../raw/crypto/HGAME2026-ezdlp-wp.md) | 模合数矩阵 DLP 可先取行列式，把 `a^k=b` 降成 `det(a)^k=det(b)`，再分模数做 Pohlig-Hellman 和 CRT。 |
| [LilacCTF2026-myblock-wp](../raw/crypto/LilacCTF2026-myblock-wp.md) | 8 轮 16-bit Feistel 的轮函数是 `GF(2^16)` 上低次多项式；用明密文样本建正向/反向基矩阵，枚举子密钥看秩下降。 |
| [LilacCTF2026-nestdlp-wp](../raw/crypto/LilacCTF2026-nestdlp-wp.md) | 二元多项式商环 DLP 可转成乘法矩阵行列式上的 p-adic DLP；恢复指数后还要用 padding 汉明重量约束解明文。 |
| [NCTF2026-encryption-wp](../raw/crypto/NCTF2026-encryption-wp.md) | pwn 后 dump `libcipher.so`，发现魔改 AES 的 S-box 被线性化；把整块加密建成 GF(2) 仿射变换矩阵再求逆。 |
| [RCTF2025-f-l-and-ag-plusplus-wp](../raw/crypto/RCTF2025-f-l-and-ag-plusplus-wp.md) | 高次数拼接关系需要快速 resultant、CRT basis 和 Lagrange interpolation；恢复比例后再 rational reconstruction 拼回 flag。 |
| [RCTF2025-suanp01y-wp](../raw/crypto/RCTF2025-suanp01y-wp.md) | 稀疏多项式满足 `hint = h1 * h0^{-1}` 于 `GF(2)[X]/(X^r-1)`；做 rational reconstruction 恢复分母，再枚举循环位移解 AES-CTR。 |
| [LilacCTF2026-lambda-m-wp](../raw/reverse/LilacCTF2026-lambda-m-wp.md) | Lambda calculus/Scott encoding 只是表达层，真实语义是 GF(2^8) 有理函数插值；把 40 个点转成齐次线性方程组。 |

## 合并与拆分结论

本页应保留为 family。DLP、Coppersmith、p-adic、GF(2) 和特殊群的首轮判断不同；但都属于“先识别代数对象，再选攻击模型”的二级路由。当前不拆小页，等某一路线积累更多 WP 后再独立成 technique。

## 常见陷阱

- 只看关键词 `LLL`/`Coppersmith`，没写根界或变量范围。
- 把群律默认成乘法群，忽略题目自定义 group law。
- Hensel lift 失败就放弃，没处理导数为 0 的重根。
- GF(2) 与整数模运算混用，矩阵方向和字节序错。
- oracle 攻击没有记录查询区间和失败响应，远程不可复现。

## 关联技巧

- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
- [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)
- [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md)
- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)
- [crypto-tooling.md](crypto-tooling.md)

## 原始资料

- [number-theory-and-algebra-attacks.md](../raw/crypto/number-theory-and-algebra-attacks.md)
- [WMCTF2025-lemonpepper-wp](../raw/crypto/WMCTF2025-lemonpepper-wp.md)
- [0xGame2020-week1-Fibonacci-wp](../raw/crypto/0xGame2020-week1-Fibonacci-wp.md)
