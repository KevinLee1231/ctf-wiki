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
updated: 2026-07-06
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
| [Bugku-apbq-rsa-ii-wp](../raw/crypto/Bugku-apbq-rsa-ii-wp.md) | 三条 `a*p+b*q` 小系数 hint 可构成缩放格，LLL 找短关系后对候选分量与 `n` 取 gcd 分解。 |
| [D3CTF2021-baby-lattice-simple-group-wp](../raw/crypto/D3CTF2021-baby-lattice-simple-group-wp.md) | 小矩阵/RSA 模数上的线性关系可转成低维格，先恢复小等价表示或秘密比值。 |
| [D3CTF2022-equivalent-wp](../raw/crypto/D3CTF2022-equivalent-wp.md) | Knapsack/子集和公钥可构造等效私钥，先取正交格再找满足奇偶和大小条件的短向量。 |
| [D3CTF2023-d3bdd-wp](../raw/crypto/D3CTF2023-d3bdd-wp.md) | LWE/RLWE dual attack 和理想格短向量是主线，先检查 PRNG 与模多项式是否破坏格结构。 |
| [D3CTF2023-d3noisy-wp](../raw/crypto/D3CTF2023-d3noisy-wp.md) | 打乱余数导致 noisy CRT，先转成子集和/背包匹配，再用格规约恢复大整数。 |
| [D3CTF2023-d3pack-wp](../raw/crypto/D3CTF2023-d3pack-wp.md) | Affine hidden subset sum 需要 completion/orthogonal lattice，先找 0/1 或 -1/0/1 短向量。 |
| [HGAME2026-ezcurve-wp](../raw/crypto/HGAME2026-ezcurve-wp.md) | ECC 横坐标 oracle 每次减去 163-bit 噪声；成对查询 `t/-t` 消去部分曲线关系，再把小误差放进 LLL。 |
| [RCTF2025-f-l-and-ag-plusplus-wp](../raw/crypto/RCTF2025-f-l-and-ag-plusplus-wp.md) | 高次数拼接关系需要快速 resultant、CRT basis 和 Lagrange interpolation；恢复比例后再 rational reconstruction 拼回 flag。 |
| [SUCTF2026-IsogenyWP](../raw/crypto/SUCTF2026-IsogenyWP.md) | CSIDH 群作用与 2-isogeny 交换，三条 2-isogenous 共享曲线高位泄露可建 CI-HNP 小根方程，用 Automated Coppersmith 恢复参数。 |

## 合并与拆分结论

本页应保留为 family。HNP、截断 LCG、LWE、Ring-LWE、orthogonal lattice 和 subset sum 都有独立构造，但共享“把有界未知转为格问题”的首轮判断。当前不拆小页，是因为 raw 仍以路线集合和 WP 案例索引为主。

## 常见陷阱

- 没估计误差界和维度就上 LLL，失败后无法判断是建模错还是参数不够。
- 把少值 error 当小噪声，导致格构造方向错误。
- Ring-LWE flatten 时搞错多项式模、系数顺序或卷积方向。
- Babai/CVP 目标向量偏移漏掉已知 bit 段。
- 解出候选后不正向复算签名、密文或 oracle 输出。

## 关联技巧

- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
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
