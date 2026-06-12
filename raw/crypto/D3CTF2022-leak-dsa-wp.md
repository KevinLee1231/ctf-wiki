# leak_dsa

## 题目简述

题目是一个标准的dsa，但泄露了k&mask的值，也就是泄露了k不连续的一些bit，将每一段连续的未知值设为$k_i$(上界为$2^{K_i}$)，可以得到

$$
k=k\&mask+\sum_i^t k_i*2^{m_i}
$$

官方仓库 https://github.com/shal10w/D3CTF-2022-crypto-d3share_leakdsa 收录了 `d3share` 与 `leak_dsa` 两道 Crypto 题的附件、exp 和 writeup，主要代码是 Python/Sage；其中 `leak_dsa` 的核心利用就是把不连续 nonce bit 泄露整理成格问题，再化归为 Hidden Number Problem。

## 解题过程

将每个k_i都看做一个未知数，则k可以表示为一个多项式，我们希望这个多项式通过简单的变换能够有一个较小的上界，这样就能够将问题转化为普通的已知msb的hnp问题了。因此我们可以使用

coppersmith的思路，构造一个格

$$
M=\begin{bmatrix}
p*2^{K_0} & 0 & \cdots & 0 \\
0 & p*2^{K_1} & \cdots & 0 \\
\vdots & \vdots & \ddots & p*2^{K_t} \\
2^{k_0}*2^{K_0} & 2^{k_1}*2^{K_1} & \cdots & 2^{k_t}*2^{K_t}
\end{bmatrix}
$$

对M使用LLL算法后可以得到较小向量$v$，以此为系数的多项式的上界是比较小的，计算1-范式后得到大多数大约为251bit左右，相当于5bit的msb leak。有一些mask会更多。计算

```
g = (M.LLL()[0,0]//2**K0)*inverse_mod(2**k0 , q) % q
```

回到dsa等式中

$$
k=m/s+dr/s
$$

$$
dr/s+m/s-k\&mask=\sum_i^t k_i*2^{m_i}
$$

$$
g*(dr/s+m/s-k\&mask)=g*\sum_i^t k_i*2^{m_i}\le ||v||_1
$$

这样，题目转化为了简单的hnp问题，解出后求得flag

## 方法总结

- 核心技巧：不要把 `k & mask` 当成连续高位泄露，而是按 mask 的连续空洞拆成多个小未知量 $k_i$，再用格规约找出一个让未知部分整体上界变小的线性组合。
- 转化路径：不连续 bit 泄露 $\rightarrow$ 多变量小根/LLL 降维 $\rightarrow$ 近似 MSB 泄露 $\rightarrow$ 标准 HNP 求私钥。
- 复用要点：类似 DSA/ECDSA nonce 只泄露若干离散 bit 时，先整理每段未知 bit 的移位和上界，再判断能否用 LLL 把它压成可处理的 HNP 实例。
