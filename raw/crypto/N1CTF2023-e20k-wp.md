# N1CTF 2023 e20k Writeup

## 题目简述

题目实现了一个基于环上短向量的证明流程。秘密向量 $\vec{s}$ 位于 $R_q=\mathbb{Z}_q[x]/(x^{256}+1)$，公开 $\vec{A}$、$t=\vec{A}\cdot\vec{s}$，并输出承诺、挑战 $c$、响应 $\vec{z}=\vec{s}c+\vec{y}$ 以及用 `MD5(str(secret))` 派生密钥加密的 flag。

随机掩码 $\vec{y}$ 的每个分量都会调用 `random.seed(prng.next())`。`ECPrng` 工作在复合模数 $N$ 上的椭圆曲线 $E:y^2=x^3+3x+7$，其状态更新是 $Q\leftarrow4Q$。攻击者可以自行提交初始点，这使“证明中的随机掩码应彼此独立”这一前提失效。

## 解题过程

### 分解结构化模数

密钥生成代码先取素数 $P$，再取满足 $2Q-1$ 也是素数的 $Q$，最终得到：

$N=P\cdot Q\cdot(2Q-1)$。

官方解法在商环 $(\mathbb{Z}/N\mathbb{Z})[x]/(x^2+s)$ 中随机取元素并计算其 $2N$ 次幂。复合模数下，不同素因子对应分量的行为可能不一致；幂结果某个系数一旦成为非平凡零因子，计算 `gcd(coefficient, N)` 就能取出一个因子。若先得到 $2Q-1$，则 $(p+1)/2=Q$，剩余因子再由整除得到。

```python
R = PolynomialRing(Zmod(N), 'x').quotient(x^2 + s)
a = R.random_element() ** (2*N)
for coefficient in a.list():
    d = gcd(ZZ(coefficient), N)
    if 1 < d < N:
        # d、(d+1)//2、N//(d*((d+1)//2)) 给出三个素因子
        break
```

### 构造固定的 ECPrng 状态

在三个素域上分别把曲线降模，求一个非零 3 阶点 $Q_i$，再分别对横、纵坐标做 CRT，得到复合模数曲线上的点 $Q$。因为 $3Q=\mathcal O$，所以 $4Q=Q$。把它作为初始状态后，每次 `next()` 都返回相同的横坐标加同一个固定偏置。

于是生成 $\vec{y}$ 的每次 `random.seed` 都收到相同种子，所有分量都变成同一个多项式 $y_0$。响应关系退化为：

$z_i=s_i c+y_0$。

### 消去掩码并恢复秘密

挑战 $c$ 在题目的商环中可逆。以第一个分量为基准，对每个响应作差即可消去 $y_0$：

$\delta_i=(z_i-z_0)c^{-1}=s_i-s_0$。

将 $s_i=s_0+\delta_i$ 代回公开关系 $\sum A_i s_i=t$，得到：

$s_0=(\sum A_i)^{-1}\left(t-\sum A_i\delta_i\right)$。

随后逐项计算 $s_i=s_0+\delta_i$，重建完整秘密向量。最后按服务端相同方式派生 AES-ECB 密钥并解密：

```python
key = md5(str(secret).encode()).digest()
flag = AES.new(key, AES.MODE_ECB).decrypt(bytes.fromhex(ct))
```

## 方法总结

攻击链由两个相互配合的问题组成：特殊形式的 $N$ 允许通过商环零因子分解；攻击者可控的椭圆曲线状态又允许选择 3 阶点，使更新映射 $Q\mapsto4Q$ 出现不动点。固定 PRNG 输出导致所有拒绝采样掩码完全相同，原本的 Ring-SIS 证明随即退化成简单的线性消元。实现密码协议时，不能只验证输入点“在曲线上”，还应验证其所属子群与阶，并且不能用可被外部状态重置的普通 PRNG 生成证明随机量。
