---
type: family
tags: [crypto, family, lattice, lwe, hnp, cvp, svp]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/lattice-and-lwe.md
  - ../raw/crypto/WMCTF2025-ishowsplit-wp.md
  - ../raw/crypto/WMCTF2025-lw3-wp.md
  - ../raw/crypto/WMCTF2025-lw5-wp.md
  - ../raw/crypto/WMCTF2025-splitmaster-wp.md
  - ../raw/crypto/HGAME2026-classic-wp.md
  - ../raw/crypto/HGAME2026-decision-wp.md
  - ../raw/crypto/NCTF2026-yqs-wp.md
  - ../raw/crypto/RCTF2025-superguess-plusplus-wp.md
  - ../raw/crypto/SUCTF2026-LatticeWP.md
updated: 2026-07-27
---

# Lattice and LWE Attacks

## 作用边界

本页是格与 LWE 攻击 family，用于判断一个 crypto 题是否应建模成 LLL/BKZ、Babai/CVP、SVP、HNP、截断 LCG、LWE/Ring-LWE、orthogonal lattice 或 subset sum。它不是单个 technique：不同路线的矩阵构造、目标向量、缩放、变量范围和成功校验都不同。

首轮不要因为出现“小误差”“部分泄露”“线性关系”就直接上 LLL。先确认 known/unknown/goal、模数、噪声或未知段大小，以及是否存在更简单的代数、PRNG 或协议 oracle 路线。

## 识别信号

- 关系形如 `A*s + e = b mod q`、`k_i` 部分泄露、低位/高位缺失、截断输出、近似倍数、subset sum、短向量或最近向量。
- 多个样本共享同一个 secret、nonce、状态或误差分布。
- 误差或未知量有边界、稀疏性、少值集合、bit 段结构或可人为选择。
- 输出能用候选 secret 正向复算验证。

## 最小证据

- 写清矩阵维度、样本数、模数、未知变量个数、变量范围和目标误差界。
- 判断是 SVP、CVP、embedding、Babai 近似、HNP 还是 subset sum。
- 记录缩放策略和预期短向量形状，避免“LLL 跑完不知道哪一行是答案”。
- 至少用小规模或 toy 参数验证构造方向。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| ECDSA/DSA nonce 有 MSB/LSB/不连续 bit 泄露 | 建 HNP/EHNP，先合并已知段和偏移，再用 CVP/Babai 或 BKZ | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| LCG/PRNG 输出被截断 | 先判断能否用递推直接恢复；不行再把未知低位作为格变量 | [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md), [prng-z3-lcg-and-timing-attacks.md](prng-z3-lcg-and-timing-attacks.md) |
| Plain LWE，误差小或 secret 稀疏/三值 | 先做 embedding/CVP 基线，再按误差分布调缩放和 block size | [crypto-tooling.md](crypto-tooling.md) |
| error 不是小噪声但只来自少数固定值 | 把“选哪个 error”转成 0/1 选择变量，而不是强行当小误差 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| Ring-LWE/Module-LWE | 先识别环、多项式基、维度和模数，再决定是否 flatten 成 plain LWE | [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md) |
| Approximate GCD、knapsack、subset sum | 先判断目标是短关系还是最近和，再转数论/代数路线 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| HSSP/AHSSP、orthogonal lattice | 先构造正交关系并检查样本数是否足够 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| Truncated LFSR / high-bit linear recurrence | 先确认递推阶数、模数是否已知和泄露高低位宽；未知模数时先恢复模数再恢复状态 | [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md) |
| 低泄露 HNP / lattice sieving | 样本数、泄露 bit 数和模数位数是否接近理论边界，是否需要 Fourier/predicate 辅助 | [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-ishowsplit-wp](../raw/crypto/WMCTF2025-ishowsplit-wp.md) | 隐藏数泄露不是连续 MSB/LSB，而是 `split_master` 给出的 20 bit 和 10 bit 不连续片段；用 EHNP 的 `Pi/Nu` 描述未知段，LLL 后 Babai CVP 恢复。 |
| [WMCTF2025-lw3-wp](../raw/crypto/WMCTF2025-lw3-wp.md) | LWE error 取三个随机模数值，不能简单线性映射成小噪声；把每行 error 选择展开为 0/1 变量，再降维并用精确 SVP/强规约恢复。 |
| [WMCTF2025-lw5-wp](../raw/crypto/WMCTF2025-lw5-wp.md) | 服务端让选手自选五个 error 值并用格 check 排除过小映射；首轮应先研究 check 约束，再利用少值 error 的选择结构恢复 secret。 |
| [WMCTF2025-splitmaster-wp](../raw/crypto/WMCTF2025-splitmaster-wp.md) | 交互 oracle 允许自选切分 `a*key mod q` 的泄露段；通过多样本相减消去公共 key 部分，把未知段和乘子关系转成 BKZ 可恢复的短向量。 |
| [HGAME2026-classic-wp](../raw/crypto/HGAME2026-classic-wp.md) | RSA `p` 高位泄露时不要暴力分解；把未知低位写成一元小根，恢复 `p` 后再处理明文中的二层古典密码提示。 |
| [HGAME2026-decision-wp](../raw/crypto/HGAME2026-decision-wp.md) | LWE/随机 chunk 判别题不能逐块解；先找两组真 LWE 样本恢复共享 secret，再用误差大小给所有 chunk 分类。 |
| [NCTF2026-yqs-wp](../raw/crypto/NCTF2026-yqs-wp.md) | 格攻击只负责恢复坐标 XOR 所需的 LWE 私钥；后半段是无效曲线小子群和 CRT，不应停在 LWE 解密。 |
| [RCTF2025-superguess-plusplus-wp](../raw/crypto/RCTF2025-superguess-plusplus-wp.md) | HNP 仅泄露 2 bit MSB 且样本数接近下界；普通 LLL 往往不够，需 lattice sieving/Fourier predicate 并接受概率性成功。 |
| [SUCTF2026-LatticeWP](../raw/crypto/SUCTF2026-LatticeWP.md) | 高位截断线性递推在未知模数场景下要先用高位格和 resultant gcd 找模数，再切回 known modulus 格恢复反馈多项式和低位状态。 |
| [DownUnderCTF2023-apbq-rsa-ii-wp](../raw/crypto/DownUnderCTF2023-apbq-rsa-ii-wp.md) | 三条 `a*p+b*q` 小系数 hint 可构成缩放格，LLL 找短关系后对候选分量与 `n` 取 gcd 分解。 |
| [D3CTF2021-baby-lattice-simple-group-wp](../raw/crypto/D3CTF2021-baby-lattice-simple-group-wp.md) | 小矩阵/RSA 模数上的线性关系可转成低维格，先恢复小等价表示或秘密比值。 |
| [D3CTF2022-equivalent-wp](../raw/crypto/D3CTF2022-equivalent-wp.md) | Knapsack/子集和公钥可构造等效私钥，先取正交格再找满足奇偶和大小条件的短向量。 |
| [D3CTF2023-d3bdd-wp](../raw/crypto/D3CTF2023-d3bdd-wp.md) | LWE/RLWE dual attack 和理想格短向量是主线，先检查 PRNG 与模多项式是否破坏格结构。 |
| [D3CTF2023-d3noisy-wp](../raw/crypto/D3CTF2023-d3noisy-wp.md) | 打乱余数导致 noisy CRT，先转成子集和/背包匹配，再用格规约恢复大整数。 |
| [D3CTF2023-d3pack-wp](../raw/crypto/D3CTF2023-d3pack-wp.md) | Affine hidden subset sum 需要 completion/orthogonal lattice，先找 0/1 或 -1/0/1 短向量。 |
| [HGAME2026-ezcurve-wp](../raw/crypto/HGAME2026-ezcurve-wp.md) | ECC 横坐标 oracle 每次减去 163-bit 噪声；成对查询 `t/-t` 消去部分曲线关系，再把小误差放进 LLL。 |
| [RCTF2025-f-l-and-ag-plusplus-wp](../raw/crypto/RCTF2025-f-l-and-ag-plusplus-wp.md) | 高次数拼接关系需要快速 resultant、CRT basis 和 Lagrange interpolation；恢复比例后再 rational reconstruction 拼回 flag。 |
| [SUCTF2026-IsogenyWP](../raw/crypto/SUCTF2026-IsogenyWP.md) | CSIDH 群作用与 2-isogeny 交换，三条 2-isogenous 共享曲线高位泄露可建 CI-HNP 小根方程，用 Automated Coppersmith 恢复参数。 |
| [0xGame2024-week3-LLL-II-wp](../raw/crypto/0xGame2024-week3-LLL-II-wp.md) | 这是一类带小误差的模线性关系：未知乘数很大，但每轮增量 $b_i$ 只有约 128 位。把模同余改写成格中的整数关系后，秘密参数与小增量共同组成异常短向量。恢复 $a$ 后，由初始化关系 $C_0\equiv a\cdot seed\pmod m$ 直接求逆即可得到 seed。 |
| [0xGame2025-week2-炽羽-wp](../raw/crypto/0xGame2025-week2-炽羽-wp.md) | 利用格中异常短的隐藏向量做 LLL 规约，并利用 CFB 的密文反馈从第二块重新同步；短字节向量与超长随机向量被整数线性组合；密文丢失原始 IV 但保留连续前置密文块。 |
| [0xGame2025-week4-逐望-wp](../raw/crypto/0xGame2025-week4-逐望-wp.md) | 本题的主线是“截断的复合随机数 → 两个完整分块 → 带小误差的曲线坐标 → 二元 Coppersmith → 相邻 EC-LCG 点”。处理类似题目时，应先从总位数和分块宽度判断哪些调用结果未被截断，再把编码误差写进公开代数关系。多元小根并不是简单地“多爆几个未知量”，其可行性依赖模数、次数、根界和所构造格的维度；本题可复现的关键参数是二元曲线方程、两个 $2^{36}$ 根界以及 `m=2` 的格构造。 |
| [ACTF2023-managerfaker-wp](../raw/crypto/ACTF2023-managerfaker-wp.md) | 利用 X.509 宽松解析产生“解析结果相同、原始编码不同”的证书，再用 LLL 求解 32 位线性滚动哈希的受限后缀碰撞；安全边界一侧按结构解析并规范化对象，另一侧却对原始字节做弱哈希时，应检查表示层歧义是否能绕过身份绑定。 |
| [ACTF2025-AAALLL-wp](../raw/crypto/ACTF2025-AAALLL-wp.md) | 利用 $X^n+1$ 根集合的逆元对，把 Partial Vandermonde 实例拆成两个半维短向量问题，再用核格与 LLL 恢复秘密；公开三元多项式在单位根或 negacyclic 环根上的部分取值，同时参数满足 $X^n=-1$，应优先检查根集合的共轭、逆元或自同构配对。 |
| [D3CTF2024-enctwice-wp](../raw/crypto/D3CTF2024-enctwice-wp.md) | 先用 AGCD 格攻击恢复隐藏公因子 $X$，再利用可重组的 `val=tag+ct2*X` 构造 padding oracle；多组大整数共享未知大因子且只有小余数、样本数受限、服务又能调整公共参数时，应检查 AGCD/LLL；解密接口区分 padding 错误时应继续考虑 oracle。 |
| [MoeCTF2024-EzPack-wp](../raw/crypto/MoeCTF2024-EzPack-wp.md) | 这不是直接对原 `key` 做贪心的普通背包题。先要准确写出指数 $s=n+\sum b_i(k_i-1)$，利用平滑子群阶完成离散对数，再对调整后的超递增权重 $w_i=k_i-1$ 从大到小贪心。两个容易被忽略的检查分别是常数项 $n$ 与 $s<\operatorname{ord}(7)$；前者保证背包目标正确，后者保证离散对数没有模阶歧义。 |
| [SekaiCTF2026-needLe-in-a-multivariate-sekai-wp](../raw/crypto/SekaiCTF2026-needLe-in-a-multivariate-sekai-wp.md) | 多变量签名的安全性不能只看维数。若大量签名暴露了同一隐藏二次型的短向量，格约简可能恢复其正交分解，使高维求解退化为若干低维表示问题。官方脚本的重点是“先恢复结构，再表示目标”，而不是直接在 144 维空间盲搜。 |
| [UMDCTF2017-free-flag-encrypter-wp](../raw/crypto/UMDCTF2017-free-flag-encrypter-wp.md) | Merkle–Hellman 的私钥超递增性质并不会自动让公开背包安全。这里的公开权重数量不大、数值尺度相对较小，子集和密度足以让 LLL 找到选择向量。利用已知 flag 格式先降维、再用子集和等式和 SHA-256 双重验证，可避免把偶然短向量误当成解。 |
| [UMDCTF2023-adi-shamirs-sharing-system-wp](../raw/crypto/UMDCTF2023-adi-shamirs-sharing-system-wp.md) | 把遮罩造成的连续未知比特段参数化，再利用过量 Shamir 份额之间的多项式一致性构造格，用 LLL 恢复小范围未知量；秘密分享阈值为 64，却给出 128 个部分损坏的份额，说明额外份额不是冗余附件，而是恢复缺失比特的约束来源。 |
| [UMDCTF2023-caterpie-wp](../raw/crypto/UMDCTF2023-caterpie-wp.md) | 将小参数 LWE 的秘密恢复写成 BDD，再用 Kannan 嵌入转成唯一短向量问题并运行 BKZ；方阵维数只有几十、模数不大、秘密和误差均为窄分布时，LWE 的理论安全性并不代表具体参数安全。 |
| [UMDCTF2023-ekans-wp](../raw/crypto/UMDCTF2023-ekans-wp.md) | 把“误差等于秘密逆序”写成置换矩阵关系，将 LWE 直接化为模线性方程；噪声、nonce 或随机量若由秘密通过公开确定性函数生成，应先检查能否移项合并，而不是立即使用通用格攻击。 |
| [UMDCTF2024-giedi-composite-wp](../raw/crypto/UMDCTF2024-giedi-composite-wp.md) | 利用 $x^{210}-1$ 的低次因子把 NTRU 格投影到三个 70 维商环，分别做 BKZ 后再用多项式 CRT 重建私钥；NTRU 参数中的维度 $N$ 为高度合数，且模多项式为 $x^N-1$ 时，应先检查能否分解成显著更低维的因子。 |
| [UMDCTF2025-cover-yourself-in-oil-wp](../raw/crypto/UMDCTF2025-cover-yourself-in-oil-wp.md) | 从公钥压缩/展开规则中提取公共核，使多变量二次方程在一个大退化子空间上降为线性系统；若“压缩”通过保留少数列、再用固定倍数恢复，其秩和核通常远低于原设计预期。 |
| [UMDCTF2025-prime-sponsorship-wp](../raw/crypto/UMDCTF2025-prime-sponsorship-wp.md) | 把复用私钥的两个 NTRU 格求交，再在交格中寻找共享的唯一短向量；同一短秘密在不同模多项式、维度或参数集下产生多个公开密钥。 |
| [UMDCTF2026-weave-wp](../raw/crypto/UMDCTF2026-weave-wp.md) | 本题的突破口是先消去公开的坐标变换，再检查错误秩是否仍落在 Gabidulin 唯一解码半径内。公开“看似混淆”的陷门矩阵不仅没有增加安全性，反而把实例直接还原为标准码字加低秩错误。实现时最容易出错的是有限域基和序列化字节序，AES-GCM 的 tag 可作为最终一致性校验。 |
| [WMCTF2020-baby-sum-wp](../raw/crypto/WMCTF2020-baby-sum-wp.md) | `babySum` 是 `Sum` 技术点的简化版：低密度子集和可以用格规约求短向量，维度降低后 BKZ 成功率和速度都会明显改善。遇到类似题目时，先确认是否存在低密度、固定汉明重量或可取补集降低重量的条件；如果有，就构造带两列约束的子集和格，用 BKZ/LLL 查找形如 0/1 向量的短行并验证求和结果。 |
| [WMCTF2020-sum-wp](../raw/crypto/WMCTF2020-sum-wp.md) | 低密度子集和的核心信号是元素数量不大、目标为 0/1 选择向量、密度低且汉明重量可利用。若原向量汉明重量很大，可以先取补集降低重量。本题把 160 个 `1` 转成 20 个 `1`，再用格基把“和为 $s$、重量为 $k$”两个条件压到最后两列，使正确选择向量成为极短向量。维度过高时使用 Zero-Forced Lattices 随机强制部分 0 坐标降维，再通过多核反复 BKZ 搜索。 |
| [WMCTF2024-k-cessation-wp](../raw/crypto/WMCTF2024-k-cessation-wp.md) | 距离密文不是直接泄露 bit，而是泄露“目标位置与中间位置取反”的约束；最高位随机翻转只影响 ASCII 的第 8 bit，低 7 bit 可保留。 |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| 有界未知满足模多项式，或 nonce/状态只泄露部分位 | [lattice-small-root-and-partial-leakage.md](lattice-small-root-and-partial-leakage.md) |
| 样本数超过环境维度，先找观测矩阵的整数左核关系 | [overcomplete-matrix-linear-relation-recovery.md](overcomplete-matrix-linear-relation-recovery.md) |
| 顺序矩阵乘积带 trace 泄露，可按前缀/后缀逐项剥离 | [ordered-matrix-product-trace-peeling.md](ordered-matrix-product-trace-peeling.md) |

## 合并与拆分结论

本页应保留为 family。HNP、截断 LCG、LWE、Ring-LWE、orthogonal lattice 和 subset sum 都有独立构造，但共享“把有界未知转为格问题”的首轮判断。小根/部分泄露、过完备线性关系和有序矩阵 trace 已有独立 technique，其余路线继续作为本页 case routing。

## 常见陷阱

- 没估计误差界和维度就上 LLL，失败后无法判断是建模错还是参数不够。
- 把少值 error 当小噪声，导致格构造方向错误。
- Ring-LWE flatten 时搞错多项式模、系数顺序或卷积方向。
- Babai/CVP 目标向量偏移漏掉已知 bit 段。
- 解出候选后不正向复算签名、密文或 oracle 输出。

## 关联技巧

- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
- [overcomplete-matrix-linear-relation-recovery.md](overcomplete-matrix-linear-relation-recovery.md)
- [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)
- [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md)
- [prng-z3-lcg-and-timing-attacks.md](prng-z3-lcg-and-timing-attacks.md)
- [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)
- [crypto-tooling.md](crypto-tooling.md)

## 原始资料

- [lattice-and-lwe.md](../raw/crypto/lattice-and-lwe.md)
- [WMCTF2025-ishowsplit-wp](../raw/crypto/WMCTF2025-ishowsplit-wp.md)
- [WMCTF2025-lw3-wp](../raw/crypto/WMCTF2025-lw3-wp.md)
- [WMCTF2025-lw5-wp](../raw/crypto/WMCTF2025-lw5-wp.md)
- [WMCTF2025-splitmaster-wp](../raw/crypto/WMCTF2025-splitmaster-wp.md)
- [HGAME2026-classic-wp](../raw/crypto/HGAME2026-classic-wp.md)
- [HGAME2026-decision-wp](../raw/crypto/HGAME2026-decision-wp.md)
- [NCTF2026-yqs-wp](../raw/crypto/NCTF2026-yqs-wp.md)
- [RCTF2025-superguess-plusplus-wp](../raw/crypto/RCTF2025-superguess-plusplus-wp.md)
- [SUCTF2026-LatticeWP](../raw/crypto/SUCTF2026-LatticeWP.md)
