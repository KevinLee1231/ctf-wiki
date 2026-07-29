# SPARROW

## 题目简述

题目实现了一个 128 位自定义分组密码。Oracle 对固定的秘密明文与密钥进行加密，但每个样本会先向密钥加入已知故障向量，再把 128 个输出比特随机打乱。不同轮 Oracle 使用不同 seed，因而内部置换和线性层也不同。

表面上未知的输出置换破坏了比特位置，实际上题目 S-box 是仿射映射。固定 seed 后，整个密码都能写成 $\operatorname{GF}(2)$ 上的仿射函数：

$$
E(k,m)=Am+Bk+C.
$$

随机排列会改变位置，却保持汉明重量。这一整数不变量足以先恢复每轮未打乱的中间向量，再联立两轮方程恢复明文和密钥。

## 解题过程

### 线性化本地加密器

服务会返回本轮 seed，因此可以在本地复现同一组轮置换。用全零明文、全零密钥得到

$$
C=E(0,0).
$$

逐个翻转明文的 128 个基向量，输出与 $C$ 异或后组成矩阵 $A$；同理逐个翻转密钥基向量得到 $B$：

```python
C = bits(encrypt(zero_key, zero_msg, seed))

A = matrix(GF(2), [
    bits(encrypt(zero_key, basis(i), seed)) + C
    for i in range(128)
]).transpose()

B = matrix(GF(2), [
    bits(encrypt(basis(i), zero_msg, seed)) + C
    for i in range(128)
]).transpose()
```

这里的 `+` 是 $\operatorname{GF}(2)$ 加法，即按位异或。

### 用汉明重量消除未知排列

令本轮不含故障的未知部分为

$$
Y=Am+Bk,
$$

第 $j$ 个已知密钥故障为 $e_j$，则打乱前向量为

$$
Y\oplus E_j,\qquad E_j=Be_j+C.
$$

虽然排列 $P_j$ 未知，但返回密文的汉明重量满足

$$
w_j=\operatorname{wt}(P_j(Y\oplus E_j))
=\operatorname{wt}(Y\oplus E_j).
$$

把未知比特写为整数变量 $y_i\in\{0,1\}$，可展开成普通整数线性方程：

$$
w_j=\operatorname{wt}(E_j)+
\sum_{i=1}^{128}(1-2E_{j,i})y_i.
$$

收集约 128 至 132 个独立故障样本后，矩阵的每一行都是 $1-2E_j$，右侧为 $w_j-\operatorname{wt}(E_j)$。在有理数或整数域上线性求解，并检查结果全为 0/1，即可恢复本轮的完整 $Y$；全程不需要猜测随机排列。

### 联立不同 seed 恢复两个秘密

再请求至少一轮不同 seed，重复上述步骤，得到

$$
\begin{aligned}
Y_1&=A_1m+B_1k,\\
Y_2&=A_2m+B_2k.
\end{aligned}
$$

把两个 128 位未知向量拼成 256 位变量，在 $\operatorname{GF}(2)$ 上求解块矩阵：

$$
\begin{bmatrix}
A_1&B_1\\
A_2&B_2
\end{bmatrix}
\begin{bmatrix}
m\\k
\end{bmatrix}
={}
\begin{bmatrix}
Y_1\\Y_2
\end{bmatrix}.
$$

若矩阵秩不足，就再取一个 seed 增加方程。恢复 `m` 和 `k` 后按服务要求提交，即可取得 flag。

完整的矩阵构造、Oracle 交互和求解代码见 [R3CTF 2024 Crypto Writeup](https://tang.cat/2024/06/10/R3CTF-2024-Crypto-Writeup.html)。正文已说明为什么 S-box 令全密码仿射化、随机排列保留什么信息，以及如何从整数方程过渡到 $\operatorname{GF}(2)$ 联立求解。

## 方法总结

本题的突破口是区分“位置未知”和“信息完全丢失”。随机 shuffle 隐藏了坐标，却保留汉明重量；已知故障又让每个重量变成关于 128 个秘密比特的独立整数线性方程。先在整数域恢复每轮中间向量，再在二元域恢复明文与密钥，是两个不同的线性代数阶段，不能混在同一个模 2 方程组里，否则每个重量只剩奇偶性，信息会大幅丢失。
