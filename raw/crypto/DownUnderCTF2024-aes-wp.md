# AES

## 题目简述

题名中的 AES 实际是 **Algebraic Eraser Scheme**。`cbkap.sage` 在 $\mathbb F_{743}$ 上构造 16 股彩色 Burau 群，公开 $\tau_i$、种子矩阵 $\kappa$、Alice 子单体生成元 $A$，以及 Alice、Bob 的公开矩阵和置换；共享秘密的 $15\times15$ 矩阵经 PBKDF2（盐为 `DownUnderCTF 2024`、迭代 $10^6$）导出 ChaCha20 密钥来加密 flag。

私钥矩阵是 $\kappa$ 的多项式，私有 braid 是公开子单体的长度 64 单词。该构造泄露了足以线性恢复 Bob 私钥矩阵（差一个标量）的关系，并允许由 Alice 的公开置换找到公开生成元的短表示；因此无需解决一般 braid 群难题。

## 解题过程

首先按官方 solver 将公开 `A` 从 Laurent 多项式矩阵映射到 $N\times S_{16}$。从 $A$ 中挑选非平凡、低阶的 $\alpha=(a,1)$，并令 Bob 公开元素的矩阵部分为 $q$。由 Algebraic Eraser 关系和私钥矩阵 $d$ 与 $\kappa$ 可交换，建立线性方程：

$$
d\,\varphi(\alpha)= (\mathrm{Bob}\cdot\alpha)_M q^{-1}d,
\qquad d\kappa=\kappa d.
$$

把两个矩阵等式逐项展开为 $\mathbb F_{743}$ 上关于 $d$ 的 $15^2$ 个未知量的线性系统；`right_kernel()` 的基向量重排为矩阵就是 `candidate_d`。标量歧义在后续的共轭表达式中抵消。

接着只在置换群中处理 Alice 的公开置换 $g$。官方 `minkwitz_1998` 对 $A$ 的置换部分建立短词表，再分解 $g$ 得到指数对 $(i,\epsilon)$，并据此计算：

```python
phi_delta = prod(A[i] ** e for i, e in i_epsilon)
phi_h_delta = phi_delta * MG((M.one(), h))
shared_secret = (candidate_d * alice[0] * phi_delta[0].inverse()
                 * candidate_d.inverse() * q * phi_h_delta[0])
```

其中 $h$ 是 Bob 的公开置换。最后将 `shared_secret` 的矩阵条目转成三位十六进制文本，执行与题目相同的 PBKDF2，再以公开 `nonce` 解 ChaCha20 密文。`solve/solve.sage` 已完整实现该流程；它枚举短置换词、求线性核和 KDF 均为求解的必要部分，本文未在本地执行它。仓库配置记录的验证值为：

```text
DUCTF{b0p_1t_br41d_1t_tw15t_1t_c0nju64t3_1t}
```

## 方法总结

代数密钥交换不能只依赖“非交换群看起来复杂”。若公开操作把秘密限制在线性可表示空间，并且同态、可交换子代数或公开置换投影带来额外方程，攻击者可把群问题降为有限域线性代数加短词分解。协议设计需要证明公开键不会泄露这种线性化关系，并避免让可恢复的置换投影成为私有 word 的替代表示。
