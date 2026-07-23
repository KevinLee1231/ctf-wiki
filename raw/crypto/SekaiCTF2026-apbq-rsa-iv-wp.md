# apbq-rsa-iv

## 题目简述

题目生成 2048 位 RSA 模数 $N=pq$，使用 $e=65537$ 加密 flag，并泄露三组

$$
h_i=a_i p+b_i q,
$$

其中 $a_i,b_i<4^{312}$。直接解多元方程代价很高，官方 `solve.sage` 使用格约简恢复足够小的系数组合，再由判别式分解 $N$。

## 解题过程

令 $X,Y,Z$ 分别对应三组 hint。官方脚本先在模 $N$ 意义下消去它们共享的大素数结构，构造：

```sage
SMOL = 4^312
SB = SMOL << 1024

F.<X,Y,Z> = QQ[]
g = vector([
    Y - h1 / h0 % N * X,
    Z - h2 / h0 % N * X,
    X^2 - h0 * X,
]) / N
```

然后枚举总次数小于 10 的 $g_0^u g_1^v g_2^{\lfloor w/2\rfloor}X^{w\bmod2}$。脚本使用

$$
c_i=\frac{B_{2i}(4^i-1)}{i}
$$

给奇次单项式做 Bernoulli 型变换，把隐藏小系数产生的近似关系组织成整数格；每个单项式再按上界 `SB` 缩放。对该格运行 Flatter/LLL 后，将短向量除回缩放矩阵，并解一个线性系统，得到候选的

$$
ab_{\text{small}}=a_i b_i
$$

或它的一个小整数比例。这里 `SMOL` 来自题目中 $a_i,b_i<4^{312}$，而左移 1024 位对应 $p,q$ 的位数；这些界决定了短向量是否可被格约简分离。

已知 $h_i=a_i p+b_i q$ 与 $N=pq$，有：

$$
h_i^2-4a_i b_iN=(a_i p-b_i q)^2.
$$

官方脚本从约简结果取出 `ab_smol = (v[-6] / v[-4]).numerator()`。由于该值可能只恢复到比例，对小范围倍数 $1\le i<1000$ 检查：

$$
\Delta_i=h_0^2-4iNab_{\text{small}}.
$$

当 $\Delta_i$ 为完全平方数时，令 $d=\sqrt{\Delta_i}$，则

$$
\gcd(h_0+d,N)
$$

会给出 $p$ 或 $q$。得到因子后验证 $pq=N$，计算

$$
d_{\mathrm{RSA}}=e^{-1}\bmod (p-1)(q-1),
$$

再求 $m=c^{d_{\mathrm{RSA}}}\bmod N$ 并转为字节。

最终明文为：

```text
SEKAI{rural_ambiance_at_sixes_and_sevens_by_6_or_7_answer_is_two_words_of_lengths_six_and_seven}
```

## 方法总结

线性组合中同时出现两个大素数并不意味着泄露安全；真正决定难度的是系数界。小系数使多个泄露式之间产生可由格约简捕获的短关系，而恒等式
$h^2-4abN=(ap-bq)^2$ 又把短关系转化为可验证的完全平方条件。
