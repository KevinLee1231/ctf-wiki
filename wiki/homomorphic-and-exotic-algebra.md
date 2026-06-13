---
type: family
tags: [crypto, family, homomorphic, exotic-algebra, paillier, elgamal, oracle]
skills: [ctf-crypto]
raw:
  - ../raw/crypto/homomorphic-and-exotic-algebra.md
updated: 2026-06-12
---

# Homomorphic and Exotic Algebra

## 作用边界

本页是同态与非常规代数 family，覆盖 Paillier/Goldwasser-Micali/ElGamal 同态 oracle、braid group、tropical semiring、矩阵/对称群密码、format-preserving encryption、BB84、差分隐私噪声、Hamming code 和其它低频代数构造。

这些内容不应作为单一 technique。首轮需要判断题目利用的是同态可塑性、oracle 单调性、特殊群不变量、编码纠错、概率噪声可抵消，还是协议参与者可被中间人控制。

## 共同识别信号

- 密文可做乘法/加法/重随机化/复制，oracle 响应泄露 bit、大小关系、合法性或部分明文。
- 代数结构不是常见整数模乘法，而是 braid、tropical semiring、矩阵群、对称群、FPE、量子密钥分发或编码系统。
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
| FPE/Feistel 小参数、对称群 cipher | 先估计状态空间，优先表查询或低位 brute force | [block-mode-misuse-family.md](block-mode-misuse-family.md), [classical-xor-and-substitution-ciphers.md](classical-xor-and-substitution-ciphers.md) |
| Hamming code、交织、差分隐私噪声 | 先恢复编码/采样模型，再通过重复查询、纠错或噪声抵消恢复明文 | [oracles-recurrences-captcha-polyglots.md](oracles-recurrences-captcha-polyglots.md) |
| BB84/QKD、双方协议交互可控 | 先建攻击者控制的消息流和基选择，再比较校验阶段能否通过 | [hash-protocol-and-oracle-attacks.md](hash-protocol-and-oracle-attacks.md) |

## 合并与拆分结论

本页应保留为 family。它承接的是低频 crypto 结构的首轮识别，而不是具体算法步骤。当前 raw 多为短案例，拆分成 Paillier、ElGamal、Braid、FPE、BB84 等小页会形成孤立节点。

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
