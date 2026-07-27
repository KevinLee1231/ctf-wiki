---
type: family
tags: [crypto, family, homomorphic, exotic-algebra, paillier, elgamal, oracle]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/homomorphic-and-exotic-algebra.md
  - ../raw/crypto/SUCTF2026-RestaurantWP.md
  - ../raw/crypto/VNCTF2026-ezov-wp.md
updated: 2026-07-27
---

# Homomorphic and Exotic Algebra

## 作用边界

本页是同态与非常规代数 family，覆盖 Paillier/Goldwasser-Micali/ElGamal 同态 oracle、braid group、tropical semiring、多变量二次签名、矩阵/对称群密码、format-preserving encryption、BB84、差分隐私噪声、Hamming code 和其它低频代数构造。

这些内容不应作为单一 technique。首轮需要判断题目利用的是同态可塑性、oracle 单调性、特殊群不变量、编码纠错、概率噪声可抵消，还是协议参与者可被中间人控制。

## 识别信号

- 密文可做乘法/加法/重随机化/复制，oracle 响应泄露 bit、大小关系、合法性或部分明文。
- 代数结构不是常见整数模乘法，而是 braid、tropical semiring、UOV/OV 二次型矩阵、矩阵群、对称群、FPE、量子密钥分发或编码系统。
- 可查询接口允许构造选择密文、选择明文、重复采样或控制协议一方。
- 解法往往靠不变量、同态变换、二分、噪声抵消、表查询或低维 brute force。

## 最小证据

- 明确代数结构和操作：密文乘法/加法、重随机化、群律、半环、矩阵、编码交织或量子基选择。
- 明确 oracle 泄露的东西：bit、大小、有效性、错误位置、重加密等价类或协议状态。
- 能用一两个选择输入证明同态或不变量存在。
- 能正向复算最终 shared secret、key、明文或 oracle 收敛过程。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| Paillier/GM/ElGamal 可塑密文和 bit/大小 oracle | 先构造同态变换，判断可否二分、逐 bit 提取或重随机化绕检查 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) |
| ElGamal 矩阵、特殊群、clock/group law | 先恢复群结构和阶，再决定 DLP、Jordan normal form 或不变量恢复 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md), [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md) |
| Braid/tropical/monotone function | 先找乘法性、偏序、单调性或 residuation，不要直接暴力私钥 | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| UOV/OV 或多变量二次签名 | 先找 oil-oil 零块、等价子空间和可固定变量；不要把它误归成普通 hash/signature oracle | [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md) |
| FPE/Feistel 小参数、对称群 cipher | 先估计状态空间，优先表查询或低位 brute force | [block-mode-misuse-family.md](block-mode-misuse-family.md), [classical-xor-and-substitution-ciphers.md](classical-xor-and-substitution-ciphers.md) |
| Hamming code、交织、差分隐私噪声 | 先恢复编码/采样模型，再通过重复查询、纠错或噪声抵消恢复明文 | [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md) |
| BB84/QKD、双方协议交互可控 | 先建攻击者控制的消息流和基选择，再比较校验阶段能否通过 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| 密文群运算可构造明文变换并触发业务反馈 | [homomorphic-ciphertext-malleability-oracles.md](homomorphic-ciphertext-malleability-oracles.md) |
| 群/环运算最终化为多项式、模根或插值问题 | [algebraic-polynomial-and-modular-root-reconstruction.md](algebraic-polynomial-and-modular-root-reconstruction.md) |
| 反馈只给比较、成功位或 timing，需要先建立谓词 | [adaptive-oracle-response-modeling.md](adaptive-oracle-response-modeling.md) |

## 合并与拆分结论

本页应保留为 family。它承接的是低频 crypto 结构的首轮识别，而不是具体算法步骤。当前 raw 多为短案例，拆分成 Paillier、ElGamal、Braid、FPE、BB84 等小页会形成孤立节点。

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [SUCTF2026-RestaurantWP](../raw/crypto/SUCTF2026-RestaurantWP.md) | Tropical semiring 验证器只检查最终等式和 rank/range，可先构造目标矩阵 `T`，再让 `A/B/P/R/S` 把未知项压成同一结果。 |
| [VNCTF2026-ezov-wp](../raw/crypto/VNCTF2026-ezov-wp.md) | UOV/OV 公钥二次型保留 oil-oil 零块结构；可恢复等价 vinegar/oil 子空间后固定 vinegar、解 oil 来伪造目标签名。 |
| [ACTF2026-arrange-in-asceding-wp](../raw/crypto/ACTF2026-arrange-in-asceding-wp.md) | CKKS 排名题要在密文 slot 中打包 128x128 两两比较，靠旋转、Chebyshev 近似和 rescale 控制乘法深度。 |
| [D3CTF2021-alice-want-flag-wp](../raw/crypto/D3CTF2021-alice-want-flag-wp.md) | ElGamal 乘法同态和长度 oracle 可逐位泄露密码，再结合短 key meet-in-the-middle。 |
| [LilacCTF2026-bootstrapping-wp](../raw/crypto/LilacCTF2026-bootstrapping-wp.md) | 同态、配对或特殊代数结构暴露运算关系，先确认可组合操作和明文空间。 |
| [RCTF2025-repairing-wp](../raw/crypto/RCTF2025-repairing-wp.md) | Pairing/ElGamal 风格密文可重随机化：同时改 `C1,C2,C3` 保持 shared key 不变，服务端未去重时可解出原 flag key。 |
| [0xGame2024-week2-Elgamal-wp](../raw/crypto/0xGame2024-week2-Elgamal-wp.md) | 漏洞来自验签输入域不完整：$r$ 既作为指数又作为模 $q$ 的底数，却没有被限制在标准区间。通过同时满足模 $q-1$ 和模 $q$ 的两个同余条件，可以把已有签名线性缩放到新消息；修复时至少要检查 $0<r<q$、$0<s<q-1$。 |
| [ACTF2025-tinyCKKS-wp](../raw/crypto/ACTF2025-tinyCKKS-wp.md) | 从 CKKS 近似解密值乘回缩放因子，恢复含噪声的精确环元素，进而解线性方程得到秘密；再用多轮误差恢复 MT19937 状态；服务返回高精度近似明文、误差远小于缩放因子，并重复使用可预测 PRNG 生成小误差时，应检查是否能恢复 $pt+e$ 和 PRNG 输出。 |
| [UMDCTF2023-noisy-bits-wp](../raw/crypto/UMDCTF2023-noisy-bits-wp.md) | 识别 Golay $(23,12,7)$ 纠错码，利用其三位纠错能力恢复每个被扰动的码字；12 位输入、23 位输出、固定多项式和每块至多三位噪声是非常强的编码参数特征。 |
| [WMCTF2022-homo-wp](../raw/crypto/WMCTF2022-homo-wp.md) | 本题核心是把简化同态加密还原成 AGCD 问题。公钥不是随机大数，而是一组共享隐藏因子 $q$ 的近似倍数；只要用格规约恢复 $q$，每个密文位都能通过 $(c\bmod q)\bmod 2$ 解出。 |

## 常见陷阱

- 看到同态加密就只想解私钥，忽略 oracle 可直接提 bit 或比较大小。
- 没先证明操作可塑性，直接套 Paillier/ElGamal 标准公式。
- 忽略重随机化或等价类，导致同一明文被误判为不同状态。
- 差分隐私和噪声题没有做重复采样统计。
- 低频代数结构不先找不变量，直接暴力私钥空间。

## 关联技巧

- [crypto-parameter-triage-family.md](crypto-parameter-triage-family.md)
- [number-theory-and-algebra-attacks.md](number-theory-and-algebra-attacks.md)
- [ecc-dlp-and-signature-attacks.md](ecc-dlp-and-signature-attacks.md)
- [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md)
- [block-mode-misuse-family.md](block-mode-misuse-family.md)
- [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md)
- [crypto-tooling.md](crypto-tooling.md)

## 原始资料

- [homomorphic-and-exotic-algebra.md](../raw/crypto/homomorphic-and-exotic-algebra.md)
- [SUCTF2026-RestaurantWP](../raw/crypto/SUCTF2026-RestaurantWP.md)
- [VNCTF2026-ezov-wp](../raw/crypto/VNCTF2026-ezov-wp.md)
