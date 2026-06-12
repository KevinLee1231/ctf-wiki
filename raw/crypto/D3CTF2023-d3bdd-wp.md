# d3bdd

## 题目简述

题目围绕 LWE/RLWE 的 dual attack 和理想格结构设计。预期路径是利用不合适的 PRNG 生成多项式，使对应理想格左核中存在容易找到的短向量，再用 dual attack 区分 LWE 并恢复 secret；非预期路径来自模多项式选择不当，存在小因子，降维后可用 CRT 和 flag 可打印性恢复结果。

## 解题过程

### 背景

LWE 主要有两类攻击方法：primal attack 和 dual attack。理论上这两类攻击复杂度接近，但 primal attack 实现更简单，实际复杂度通常低于 dual attack，所以 CTF 中常用 primal attack 而不是 dual attack。不过近期带 FFT distinguisher 的 dual attack[[1,2]] 理论复杂度略低于 primal attack，尽管相关工作仍有一些问题[[3]]。因此我想借这道 CTF 题介绍 dual attack。

dual attack 与 primal attack 的一个区别是，dual attack 会把 LWE 问题规约到 aSVP 问题，而不是 uSVP 问题。如果某个格具有特殊性质，使它的左核中存在容易找到的极短向量，那么 dual attack 的复杂度会远小于 primal attack，这也是本题的主要思路。

本题主要分成两部分：PRNG，以及理想格上的 BDD 问题。我的预期是，不合适的 PRNG 生成的多项式会带来容易找到的极短向量，然后利用 dual attack 解决 BDD 问题。

### 非预期解

比赛中解出本题的队伍没有采用预期解法。

原因在于选择多项式环的模多项式时使用了不合适的多项式。这个多项式有很多小因子，尤其是其中两个因子；RLWE 问题模这些因子时只需要规约 256 维格，而且噪声很小，因此可以直接求解。由此可以通过 CRT 算法得到部分值。虽然另一个值无法直接得到，因为

格维数达到 512，但由于目标值是 flag，是可打印且有意义的字符串，选手可以通过猜测单词并校验 hash 判断 flag 是否正确。

非预期出现的主要原因是模多项式选择不当。本应选择更合适的多项式作为模多项式。

预期解可以同时解决这两种情况。

### 预期解

### 对偶攻击（Dual Attack）

先介绍 dual attack 的流程。

dual attack 可以判断一对 `(A,b)` 是否为 LWE 实例。如果它符合 LWE 分布，则有相应关系成立。

现在寻找两个满足条件的短向量。

在式 (1) 两边同时乘以相应向量，可以得到新的关系。

由于相关误差项很小，对应组合结果也会很小。

但如果这对 `(A,b)` 不是 LWE 实例，那么该值应接近均匀分布，期望约在相应模数范围内。借此可以解决 decision-LWE 问题。

如何利用这种攻击得到 secret？

可以枚举 `s` 的一部分，例如令 `s = (s1 | s2)`，枚举 `s1` 的取值。

如果猜测的 `s1` 正确，剩余构造就是 LWE 实例；反之则不是。因此可以通过枚举恢复完整 secret。

### 理想格

如何找到这样的短向量？首先需要观察格 `A` 的形状。如果模多项式为某一形式，格形态如下：

$$
\begin{pmatrix}
-a_{n-2} & \cdots & -a_1 \\
-a_{n-1} & \cdots & -a_2 \\
\vdots & \ddots & \vdots \\
a_{n-3} & \cdots & a_0
\end{pmatrix}
$$

如果模多项式换成另一种形式，格形态如下：

$$
\begin{pmatrix}
a_0 & a_{n-1} & a_{n-2} & \cdots & a_1 \\
a_1 & a_0 & a_{n-1} & \cdots & a_2 \\
\vdots & \vdots & \vdots & \ddots & \vdots \\
a_{n-1} & a_{n-2} & a_{n-3} & \cdots & a_0
\end{pmatrix}
$$

它们很相似，并且都有一个重要特征：每一列都具有循环结构，因此能找到很多相同模式。

例如某个模式在前两行中只有靠近对角线的第二列不匹配，其他位置都符合这个模式。

也就是说，如果能找到一个对任意 `n` 都满足对应关系的向量，

那么它与 `A` 的前两行相乘后，结果中几乎每一维都会是 0，只有一维非 0。因此目标就是找到满足类似式 (7) 的向量，再用 dual attack 解题。

### PRNG

可以发现这个 PRNG 是一个模 `m` 的线性生成器。

$$
a_{n+17}=\sum_{i=0}^{16}a_{n+i}f_i \pmod m
$$

令相应变量为某个值，则有：

$$
\sum_{i=0}^{17}a_{n+i}f_i=0 \pmod m
$$

这个 `f` 乘以 `A` 的前 17 行，已经可以让结果的大多数维度为 0。但题目中的 `f` 是随机选取的，数值很大，所以还需要更小的 `f`。

为了解决这个问题，可以多取几个 `n` 并把式 (9) 相加，即：

$$
\sum_{j=0}^{k}\left(c_j\sum_{i=0}^{17}a_{n+i+j}f_i\right)
=
\sum_{i=0}^{k+17}a_i
\left(\sum_{j=\max(0,i-k)}^{\min(17,i)}f_jc_{i-j}\right)
$$

其中部分变量可选，因此可以构造如下格来寻找小向量。

在本题参数下，`k` 取到约 80 时效果较好。但这也意味着需要枚举 80 bit，这不可接受。不过题目中的 `s` 并不是随机值，它带有长度为 9 字节（72 bit）的 flag 头 `antd3ctf{`，恰好这个 flag 头很长，因此实际不需要枚举太多。

**m 不是 q？**

这里遇到另一个重要问题。上面的计算建立在模 `m` 下，而 LWE 的模数是 `q`，并且 `m % q` 也很大。如果不解决这个问题，上面得到的向量无法使用。

尝试去掉上面公式中的 `mod m`，可以得到：

其中某项是一个向量。由于 `A` 小于 `m`，`g` 只会比 `f'` 稍大，最多是 `f'` 的 `k` 倍（`k` 是 `f'` 的维数），但实际中几乎不可能大到这个程度。

另外，在去年的 `leak_dsa`（D3CTF 2022 的一道 Crypto 题）中，我们用过一个技巧：乘上一个 magic value 来平衡某一项。

构造如下格：

规约后可以得到一个满足条件的短向量。

理论上，用这种方式得到的 `q_1` 和 `q_2` 会在某个范围附近，但本题选择的 `m` 比较特殊，得到 `q_1 = 5049`、`q_2 = -6683`。它们比预期结果小得多，所以 dual attack 的成功率会更高。即使得到的结果大小接近理论范围，也仍然可以在理论上用 dual attack 求解，但可能需要更多向量、更高维数，或者使用 sieving、FFT distinguisher 等更多技巧。

回到原始公式。

$$
\begin{aligned}
A_2s_2+e&=b' \pmod q,\\
f'A_2s_2+f'e&=mgs_2+f'e=f'b' \pmod q,\\
q_1gs_2+q_2f'e&=q_2f'b' \pmod q.
\end{aligned}
$$

最后在实验中，如果猜测正确，方程右侧的值大约是 `q/100`，约为均匀选择期望值的 `1/50`，因此可以以很高概率计算出 flag。

由于成功率不是 100%，我额外给出了 flag 的 hash，方便选手检查 flag 是否正确。

### 参考资料

[1] Guo-Johansson 的 ASIACRYPT 2021 工作给出了带 FFT distinguisher 的更快 dual lattice attack 框架，说明如何用短向量把 LWE 样本转成可区分分布。本题借用的是“找到短左核向量后做 dual distinguisher”的思路。

[2] MATZOV 的 LWE 安全报告整理了改进 dual attack 的成本估计和参数影响，用于判断 dual attack 在某些参数下可能优于 primal attack。

[3] Ducas-Pulles 讨论 dual-sieve attack 的实际可行性和潜在问题，提醒这类理论复杂度降低不一定直接等价于 CTF 参数下的稳定求解。

[1] Q. Guo 和 T. Johansson，《用于求解 LWE 并应用于 CRYSTALS 的更快 dual lattice attack》，ASIACRYPT 2021，doi: 10.1007/978-3-030-92068-5_2。

[2] MATZOV，2022，《LWE 安全性报告：改进的 dual lattice attack》，Zenodo，https://doi.org/10.5281/zenodo.6412487。

[3] L. Ducas 和 L. N. Pulles，《针对 LWE 的 dual-sieve attack 是否真的有效？》。

s://eprint.iacr.org/2023/302

## 方法总结

- 核心技巧：LWE dual attack、理想格循环结构、PRNG 线性关系构造短向量、模多项式小因子导致降维非预期。
- 识别信号：RLWE/ideal lattice 题中如果模多项式可分解出低维因子，应先检查 CRT 降维；如果设计目标是 dual attack，则看左核是否因 PRNG 或循环结构出现异常短向量。
- 复用要点：参考文献提供的是 dual attack 的理论框架和风险边界；实际 WP 必须保留本题的 PRNG 线性关系、magic value 平衡和 flag hash 校验这些具体落点。
