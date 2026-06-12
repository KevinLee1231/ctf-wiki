---
type: family
tags: [crypto, family, triage]
skills: [ctf-crypto]
updated: 2026-06-11
---

# Crypto Parameter and Oracle Triage

## 适用场景

密文、参数、签名、oracle、PRNG 输出或自定义数学协议同时出现，首轮还不能确定是 RSA、ECC、格、PRNG、分组模式还是哈希协议。

本页是 WP 摄入后的 pivot 页：它不按比赛或题目组织，而是把 raw WP 中重复出现的识别信号压缩成下一跳判断表。

## 识别信号

- 题面给出 n/e/c、曲线参数、签名、seed、oracle、PoW、验证码或自定义方程。
- 参数规模、可控查询或错误反馈能决定下一跳攻击族。
- 同一 WP 中可能同时出现数论和工程包装；先抽出数学关系再考虑工具。

## 最小证据

- 至少有一组可复算的参数、可重放的输入输出、可观察的错误/状态差异，或可定位的文件/交互边界。
- 能说明当前题更接近下表哪个下一跳技巧页，而不是只按比赛名或题名判断。
- 若多个变体同时出现，先选择能最小验证的信号；失败后再按相邻技巧 pivot。

## 解法骨架

1. 列出 known / unknown / goal 和所有可复算方程。
2. 按参数形态进入 RSA、ECC、格、PRNG、分组模式、哈希协议或代数页。
3. 做一个最小验证：因数分解、小根、状态恢复、oracle 差异或明密文复算。
4. 若失败，回到参数规模和泄露边界，换相邻技巧页。

## 关键变体

| 下一跳 | 触发信号 | WP 数 |
|---|---|---:|
| [block-mode-misuse-family.md](block-mode-misuse-family.md) | AES/分组模式、IV、padding、MAC 或加解密 oracle。 | 5 |
| [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) | 曲线/DLP/签名/同源/配对关系。 | 12 |
| [exotic-secret-sharing-rabin-and-polynomials.md](exotic-secret-sharing-rabin-and-polynomials.md) | secret sharing、Rabin、扰动多项式或可替换私钥结构。 | 1 |
| [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) | 哈希、PoW、验证码、协议校验或弱比较。 | 5 |
| [homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md) | 同态、配对或特殊代数结构。 | 2 |
| [lattice-and-lwe.md](lattice-and-lwe.md) | 格规约、小根、近似约束或高维线性关系。 | 8 |
| [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) | PRNG seed、状态恢复、MT/LCG/LFSR 输出泄露。 | 9 |
| [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) | 数论、代数方程、枚举边界或自定义数学结构。 | 8 |
| [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) | RSA 模数、素数结构、小根、低指数或 oracle。 | 14 |

## WP 案例沉淀

本节只抽取 raw WP 中可复用的识别信号和下一跳，不替代原始题解正文。

| Raw WP | 复用信号 | 下一跳 |
|---|---|---|
| [ACTF2026-arrange-in-asceding-wp](../raw/crypto/ACTF2026-arrange-in-asceding-wp.md) | 数论/代数方程是主约束，先化简等式、枚举边界并做正向复算。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [ACTF2026-inverse-pow-wp](../raw/crypto/ACTF2026-inverse-pow-wp.md) | 数论/代数方程是主约束，先化简等式、枚举边界并做正向复算。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [ACTF2026-ohmycaptcha-wp](../raw/crypto/ACTF2026-ohmycaptcha-wp.md) | 哈希、验证码、PoW 或协议校验可被降维，先确认碰撞、弱比较或 oracle。 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) |
| [ACTF2026-pandora-wp](../raw/crypto/ACTF2026-pandora-wp.md) | 数论/代数方程是主约束，先化简等式、枚举边界并做正向复算。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [Bugku-alkane-wp](../raw/crypto/Bugku-alkane-wp.md) | 数论/代数方程是主约束，先化简等式、枚举边界并做正向复算。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [Bugku-apbq-rsa-ii-wp](../raw/crypto/Bugku-apbq-rsa-ii-wp.md) | RSA 参数或素数结构异常，先列 known / unknown / goal，再判断分解、小根、低指数或 oracle。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [Bugku-Fibonacci-RSA-wp](../raw/crypto/Bugku-Fibonacci-RSA-wp.md) | RSA 参数或素数结构异常，先列 known / unknown / goal，再判断分解、小根、低指数或 oracle。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [HGAME2026-babyrsa-wp](../raw/crypto/HGAME2026-babyrsa-wp.md) | RSA 参数或素数结构异常，先列 known / unknown / goal，再判断分解、小根、低指数或 oracle。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [HGAME2026-classic-wp](../raw/crypto/HGAME2026-classic-wp.md) | 数论/代数方程是主约束，先化简等式、枚举边界并做正向复算。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [HGAME2026-decision-wp](../raw/crypto/HGAME2026-decision-wp.md) | 数论/代数方程是主约束，先化简等式、枚举边界并做正向复算。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [HGAME2026-eezzdlp-wp](../raw/crypto/HGAME2026-eezzdlp-wp.md) | DLP、曲线、同源、配对或签名 nonce 是核心，先确认群阶、基点和标量关系。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [HGAME2026-ezcurve-wp](../raw/crypto/HGAME2026-ezcurve-wp.md) | DLP、曲线、同源、配对或签名 nonce 是核心，先确认群阶、基点和标量关系。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [HGAME2026-ezdlp-wp](../raw/crypto/HGAME2026-ezdlp-wp.md) | DLP、曲线、同源、配对或签名 nonce 是核心，先确认群阶、基点和标量关系。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [HGAME2026-ezrsa-wp](../raw/crypto/HGAME2026-ezrsa-wp.md) | RSA 参数或素数结构异常，先列 known / unknown / goal，再判断分解、小根、低指数或 oracle。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [HGAME2026-flux-wp](../raw/crypto/HGAME2026-flux-wp.md) | 哈希、验证码、PoW 或协议校验可被降维，先确认碰撞、弱比较或 oracle。 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) |
| [LilacCTF2026-bootstrapping-wp](../raw/crypto/LilacCTF2026-bootstrapping-wp.md) | 同态、配对或特殊代数结构暴露运算关系，先确认可组合操作和明文空间。 | [homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md) |
| [LilacCTF2026-myblock-wp](../raw/crypto/LilacCTF2026-myblock-wp.md) | 分组模式、IV、padding 或加解密 oracle 可被复用，先固定可控明密文差异。 | [block-mode-misuse-family.md](block-mode-misuse-family.md) |
| [LilacCTF2026-myrsa-wp](../raw/crypto/LilacCTF2026-myrsa-wp.md) | RSA 参数或素数结构异常，先列 known / unknown / goal，再判断分解、小根、低指数或 oracle。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [LilacCTF2026-nestdlp-wp](../raw/crypto/LilacCTF2026-nestdlp-wp.md) | DLP、曲线、同源、配对或签名 nonce 是核心，先确认群阶、基点和标量关系。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [LilacCTF2026-noisy-forest-wp](../raw/crypto/LilacCTF2026-noisy-forest-wp.md) | PRNG/seed/状态泄露，先确定生成器类型、泄露位宽和可观测轮数。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [NCTF2026-encryption-wp](../raw/crypto/NCTF2026-encryption-wp.md) | 分组模式、IV、padding 或加解密 oracle 可被复用，先固定可控明密文差异。 | [block-mode-misuse-family.md](block-mode-misuse-family.md) |
| [NCTF2026-ez-rsa-wp](../raw/crypto/NCTF2026-ez-rsa-wp.md) | RSA 参数或素数结构异常，先列 known / unknown / goal，再判断分解、小根、低指数或 oracle。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [NCTF2026-hard-rsa-wp](../raw/crypto/NCTF2026-hard-rsa-wp.md) | RSA 参数或素数结构异常，先列 known / unknown / goal，再判断分解、小根、低指数或 oracle。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [NCTF2026-rng-game-wp](../raw/crypto/NCTF2026-rng-game-wp.md) | PRNG/seed/状态泄露，先确定生成器类型、泄露位宽和可观测轮数。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [NCTF2026-yqs-wp](../raw/crypto/NCTF2026-yqs-wp.md) | 数论/代数方程是主约束，先化简等式、枚举边界并做正向复算。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [RCTF2025-f-l-and-ag-plusplus-wp](../raw/crypto/RCTF2025-f-l-and-ag-plusplus-wp.md) | 约束可转成格规约、小根或近似关系，先估变量界和模数规模。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [RCTF2025-repairing-wp](../raw/crypto/RCTF2025-repairing-wp.md) | DLP、曲线、同源、配对或签名 nonce 是核心，先确认群阶、基点和标量关系。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [RCTF2025-suanhash-wp](../raw/crypto/RCTF2025-suanhash-wp.md) | 哈希、验证码、PoW 或协议校验可被降维，先确认碰撞、弱比较或 oracle。 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) |
| [RCTF2025-suanp01y-wp](../raw/crypto/RCTF2025-suanp01y-wp.md) | 哈希、验证码、PoW 或协议校验可被降维，先确认碰撞、弱比较或 oracle。 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) |
| [RCTF2025-superguess-plusplus-wp](../raw/crypto/RCTF2025-superguess-plusplus-wp.md) | 约束可转成格规约、小根或近似关系，先估变量界和模数规模。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [RCTF2025-yet-another-mt-game-wp](../raw/crypto/RCTF2025-yet-another-mt-game-wp.md) | PRNG/seed/状态泄露，先确定生成器类型、泄露位宽和可观测轮数。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [RCTF2025-yet-another-shuffled-mt-game-wp](../raw/crypto/RCTF2025-yet-another-shuffled-mt-game-wp.md) | PRNG/seed/状态泄露，先确定生成器类型、泄露位宽和可观测轮数。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [SU_AESWP](../raw/crypto/SU_AESWP.md) | 分组模式、IV、padding 或加解密 oracle 可被复用，先固定可控明密文差异。 | [block-mode-misuse-family.md](block-mode-misuse-family.md) |
| [SU_IsogenyWP](../raw/crypto/SU_IsogenyWP.md) | DLP、曲线、同源、配对或签名 nonce 是核心，先确认群阶、基点和标量关系。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [SU_LatticeWP](../raw/crypto/SU_LatticeWP.md) | 约束可转成格规约、小根或近似关系，先估变量界和模数规模。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [SU_PrngWP](../raw/crypto/SU_PrngWP.md) | PRNG/seed/状态泄露，先确定生成器类型、泄露位宽和可观测轮数。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [SU_RestaurantWP](../raw/crypto/SU_RestaurantWP.md) | 分组模式、IV、padding 或加解密 oracle 可被复用，先固定可控明密文差异。 | [block-mode-misuse-family.md](block-mode-misuse-family.md) |
| [SU_RSAWP](../raw/crypto/SU_RSAWP.md) | RSA 参数或素数结构异常，先列 known / unknown / goal，再判断分解、小根、低指数或 oracle。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [VNCTF2026-ezov-wp](../raw/crypto/VNCTF2026-ezov-wp.md) | 分组模式、IV、padding 或加解密 oracle 可被复用，先固定可控明密文差异。 | [block-mode-misuse-family.md](block-mode-misuse-family.md) |
| [VNCTF2026-hd-is-what-wp](../raw/crypto/VNCTF2026-hd-is-what-wp.md) | DLP、曲线、同源、配对或签名 nonce 是核心，先确认群阶、基点和标量关系。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [VNCTF2026-math-rsa-wp](../raw/crypto/VNCTF2026-math-rsa-wp.md) | RSA 参数或素数结构异常，先列 known / unknown / goal，再判断分解、小根、低指数或 oracle。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [VNCTF2026-numberguesser-wp](../raw/crypto/VNCTF2026-numberguesser-wp.md) | PRNG/seed/状态泄露，先确定生成器类型、泄露位宽和可观测轮数。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [VNCTF2026-schnorr-wp](../raw/crypto/VNCTF2026-schnorr-wp.md) | DLP、曲线、同源、配对或签名 nonce 是核心，先确认群阶、基点和标量关系。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [D3CTF2019-babyecc-wp](../raw/crypto/D3CTF2019-babyecc-wp.md) | 曲线参数和 ECC 群运算是核心，先确认曲线阶、基点和标量关系。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [D3CTF2019-bivariate-wp](../raw/crypto/D3CTF2019-bivariate-wp.md) | RSA 素数中间位泄露可写成二元小根，先估计未知高低位界并构造 Coppersmith 格。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [D3CTF2019-common-wp](../raw/crypto/D3CTF2019-common-wp.md) | 共模 RSA 小私钥指数变体，先把 Wiener/Guo 方程转成低维格再恢复 phi。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [D3CTF2019-noise-wp](../raw/crypto/D3CTF2019-noise-wp.md) | 带噪声取模 oracle 需要维护候选区间，构造固定商查询把随机噪声转成强收缩。 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| [D3CTF2019-sign2win-wp](../raw/crypto/D3CTF2019-sign2win-wp.md) | DSA/ECDSA 签名或 nonce 关系异常，先恢复签名方程中的随机数/私钥约束。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [D3CTF2021-alice-want-flag-wp](../raw/crypto/D3CTF2021-alice-want-flag-wp.md) | ElGamal 乘法同态和长度 oracle 可逐位泄露密码，再结合短 key meet-in-the-middle。 | [homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md) |
| [D3CTF2021-baby-lattice-simple-group-wp](../raw/crypto/D3CTF2021-baby-lattice-simple-group-wp.md) | 小矩阵/RSA 模数上的线性关系可转成低维格，先恢复小等价表示或秘密比值。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [D3CTF2021-easy-curve-wp](../raw/crypto/D3CTF2021-easy-curve-wp.md) | 曲线/DLP 参数是主约束，先确认群阶、基点、映射和可降维关系。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [D3CTF2022-d3bug-wp](../raw/crypto/D3CTF2022-d3bug-wp.md) | LFSR 输出直接泄露移出位，剩余低位可转成 GF(2) 线性方程或 SAT/Z3。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [D3CTF2022-d3factor-wp](../raw/crypto/D3CTF2022-d3factor-wp.md) | Prime Power RSA 的指数差满足小根方程，先在 p 的高次幂模意义下构造 Coppersmith。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [D3CTF2022-d3qcg-wp](../raw/crypto/D3CTF2022-d3qcg-wp.md) | 二次同余生成器只泄露高位，先把未知低位建成二元小根再恢复 PRNG 状态。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [D3CTF2022-d3share-wp](../raw/crypto/D3CTF2022-d3share-wp.md) | 扰动多项式密钥共享可从多节点私钥插值和格约束中恢复等效主多项式。 | [exotic-secret-sharing-rabin-and-polynomials.md](exotic-secret-sharing-rabin-and-polynomials.md) |
| [D3CTF2022-equivalent-wp](../raw/crypto/D3CTF2022-equivalent-wp.md) | Knapsack/子集和公钥可构造等效私钥，先取正交格再找满足奇偶和大小条件的短向量。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [D3CTF2022-leak-dsa-wp](../raw/crypto/D3CTF2022-leak-dsa-wp.md) | DSA/ECDSA nonce 或签名泄露进入格恢复，先写出每个签名的线性近似关系。 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| [D3CTF2023-d3bdd-wp](../raw/crypto/D3CTF2023-d3bdd-wp.md) | LWE/RLWE dual attack 和理想格短向量是主线，先检查 PRNG 与模多项式是否破坏格结构。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [D3CTF2023-d3noisy-wp](../raw/crypto/D3CTF2023-d3noisy-wp.md) | 打乱余数导致 noisy CRT，先转成子集和/背包匹配，再用格规约恢复大整数。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [D3CTF2023-d3pack-wp](../raw/crypto/D3CTF2023-d3pack-wp.md) | Affine hidden subset sum 需要 completion/orthogonal lattice，先找 0/1 或 -1/0/1 短向量。 | [lattice-and-lwe.md](lattice-and-lwe.md) |
| [D3CTF2023-d3sys-wp](../raw/crypto/D3CTF2023-d3sys-wp.md) | CTR-SM4 token 可篡改后进入 CRT-RSA partial key exposure，重点估泄露位和 Coppersmith bound。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |
| [D3CTF2025-d3fnv-wp](../raw/crypto/D3CTF2025-d3fnv-wp.md) | FNV 哈希可视为未知 key 上的小系数多项式求值，先用格恢复 key 幂向量信息。 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) |
| [D3CTF2025-d3guess-wp](../raw/crypto/D3CTF2025-d3guess-wp.md) | 带噪声猜数反馈要用概率二分收集输出，再恢复 MT19937 状态预测后续随机数。 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| [D3CTF2025-d3sys-p2-2-0-wp](../raw/crypto/D3CTF2025-d3sys-p2-2-0-wp.md) | RSA/CRT 因子有已知线性近似且修正项很小，优先建 partial factoring 小根模型。 | [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md) |

## 常见陷阱

- 把 WP 当完整题解目录使用，没有提取可迁移的触发信号。
- 只按题名命中下一跳，忽略源码、参数规模、可控输入和错误反馈。
- 已有技巧页能覆盖时仍新建单题页面，导致 wiki 退化成 writeup 列表。

## 关联技巧

- [block-mode-misuse-family.md](block-mode-misuse-family.md)
- [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)
- [exotic-secret-sharing-rabin-and-polynomials.md](exotic-secret-sharing-rabin-and-polynomials.md)
- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)
- [homomorphic-and-exotic-algebra.md](homomorphic-and-exotic-algebra.md)
- [lattice-and-lwe.md](lattice-and-lwe.md)
- [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md)
- [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)
- [rsa-specialized-structures-and-oracles.md](rsa-specialized-structures-and-oracles.md)

## 原始资料

本页原始资料以“WP 案例沉淀”表中的 raw WP 链接为准，共 64 篇。