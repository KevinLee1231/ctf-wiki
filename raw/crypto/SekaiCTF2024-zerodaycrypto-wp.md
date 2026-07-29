# zerodaycrypto

## 题目简述

题目把 flag 花括号内的 $32$ 字节解释为整数 $K$，取模数：

$$
p=2^{255}-19,
$$

并泄漏连续逆元的高位：

```python
[pow(K + i, -1, p) >> 170 for i in range(14)]
```

令 $\Delta=2^{170}$，第 $i$ 个完整逆元可写成：

$$
(K+i)^{-1}=A_i\Delta+x_i,\qquad 0\le x_i<\Delta,
$$

其中 $A_i$ 是已知高位，$x_i$ 是未知低位。因此有模方程：

$$
(K+i)(A_i\Delta+x_i)\equiv1\pmod p.
$$

这是 adversarial modular inverse hidden number problem。题目给出的提示 PDF 定义了 super beetle gamer（SBG）多项式空间，用于筛掉传统多变量 Coppersmith 格中的弱多项式。

## 解题过程

### SBG 空间中哪些信息有用

提示论文把每行形如

$$
(x_i,ix_i,\ldots,i^{d-1}x_i,1,i,\ldots,i^{k-d-1})
$$

的 $k\times k$ 矩阵称为 $(k,d)$-beetleshell，其行列式称为 $(k,d)$-beetle；当 $k\ge2d-2$ 时称为 super beetle。所有 super beetle 的线性张成空间记作 SBG。

解题真正需要的结论是：

1. SBG 对微分和变量平移封闭，所以可先构造齐次基，再代入 $x_i+A_i\Delta$。
2. $n$ 个变量、恰为 $d$ 次的分量维数为：

$$
\dim(\mathrm{SBG}_n^d)=
\begin{cases}
1,&d=0,\\
n,&d=1,\\
\binom{n-d+2}{d},&2\le d\le\frac{n+2}{2},\\
0,&\text{其他情况}.
\end{cases}
$$

3. 总维数为 $\dim(\mathrm{SBG}_n)=F_{n+3}-1$。取 $n=10$ 时正好是 $232$，可构造一个实际可约化的 $232\times232$ 格。

### 从逆元方程构造“好”多项式

把十个方程中的共同未知量 $K$ 通过行列式消去。对一个 $d$ 次多项式，如果在真实小根处可被 $p^{d-1}$ 整除，就称它为“好”多项式。beetleshell 的列变换能让若干列分别带上因子 $p$，同时通过 Vandermonde 型关系降低行列式的实际次数，因此 SBG 基中的多项式恰好提供强同余关系。

十个样本可构造：

| 次数 | 多项式数量 | 小根模数 |
|---:|---:|---:|
| $6$ | $1$ | $p^5$ |
| $5$ | $21$ | $p^4$ |
| $4$ | $70$ | $p^3$ |
| $3$ | $84$ | $p^2$ |
| $2$ | $45$ | $p$ |
| $1$ | $10$ | $1$ |
| $0$ | $1$ | $1$ |

合计 $232$ 个基元素。官方实现没有直接反复计算行列式，而是用 Vandermonde 行列式的乘积差形式生成相同多项式，减少符号计算开销。

### 建立并约化 232 维格

先忽略已知高位，生成 $\mathrm{SBG}_{10}$ 的 $232$ 个齐次基元素，再执行变量平移：

$$
x_i\longmapsto A_i\Delta+x_i.
$$

把展开后的基写成系数矩阵，只保留主元列即可得到方阵。不同次数关系分别只保证模 $p^{d-1}$ 为零，所以把各行乘以相应的 $p$ 幂，统一到模 $p^5$。随后按每个齐次基元素在 $|x_i|<\Delta$ 下的大小对列加权。

官方实现的核心流程为：

```python
N, Delta, p = 10, 2**170, 2**255 - 19
PP = p ** (N // 2)

# tups：232 个 SBG 基元素及其可整除的 p 幂
M = coefficient_matrix([
    t.eqn(*(A[i] * Delta + x[i] for i in range(N)))
    for t in tups
])
M = M[:, M.pivots()]

# 将不同强度的同余式统一到 p^5，并按小根界平衡列
M2 = normalize_rows_to_modulus(M, PP, tups)
W = diagonal_matrix(sbg_basis_bounds(tups, Delta))
reduced = flatter(M2 * W) * W**-1
```

这里 `normalize_rows_to_modulus` 和 `sbg_basis_bounds` 是对 notebook 中矩阵缩放代码的语义化概括；真正的格约化使用 `flatter -rhf 1.03`，主要耗时约一分钟。

### 解出小根并还原 flag

约化后得到的多项式都来自“好”关系，官方解法无需再跑 Gröbner basis，而是直接解线性系统，取最后 $N+1$ 个坐标恢复 $x_i$ 与齐次常数。由第一个完整逆元：

$$
K=(A_0\Delta+x_0)^{-1}\pmod p.
$$

最后验证全部 $14$ 条泄漏：

```python
assert [pow(K + i, -1, p) >> 170 for i in range(14)] == A
print(b"SEKAI{" + K.to_bytes(32, "big") + b"}")
```

官方 notebook 的恢复结果为：

```text
SEKAI{pls_help_me_write_the_paper_kthx}
```

notebook 提到的[既有 adversarial MIHNP 构造](https://ieeexplore.ieee.org/document/10089839)在泄漏最高三分之一比特时需要更多样本和约 $46441$ 维格；本题的关键改进不是照搬该格，而是只保留属于 SBG 的强多项式，将问题压到 $232$ 维。

## 方法总结

- 核心技巧：把连续逆元高位泄漏写成多变量小根问题，利用 SBG 空间筛选满足“次数 $d$、模 $p^{d-1}$”的强多项式，再做加权格约化。
- 识别信号：同一秘密的连续模逆元只泄漏高位，传统 HNP 样本不足，但消元后能得到大量 Vandermonde 型行列式关系。
- 复用要点：Coppersmith 格的质量取决于多项式是否真正提供足够高的模幂，而不是单纯堆叠乘积；构造前应分析线性相关性、空间维数、平移封闭性和每一行的同余强度。
