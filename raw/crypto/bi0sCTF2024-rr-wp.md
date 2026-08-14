# bi0sCTF 2024 - rr

## 题目简述

题目把 flag 作为整数 $m$，随机选择 20 个系数 $k_i$，并公开以下数据：

$$
c_1=\left(\sum_{i=0}^{19}k_im^i\right)^{127}\bmod n,
$$

$$
c_2=m^{65537}\bmod n,
$$

以及每个系数的指数形式

$$
K_i=69^{k_i}\bmod r^{i+2}.
$$

其中 $r$ 是公开大素数，$k_i<r^{i+1}$。第一步要利用模素数幂单位群上的显式同态恢复所有 $k_i$；第二步把两个密文写成关于同一消息的多项式，并通过 Franklin-Reiter 型多项式最大公因式恢复 $m$。

## 解题过程

### 在模 $r^s$ 上恢复指数

直接对 $K_i=69^{k_i}\bmod r^{i+2}$ 求通用离散对数并不合算。设当前模数指数为 $s=i+2$，官方 solver 定义

$$
L_s(x)=\frac{x^{r^{s-1}(r-1)}\bmod r^{2s+1}-1}{r^s}.
$$

指数 $r^{s-1}(r-1)$ 先消去单位群中的有限阶部分，使结果落入以 1 开头的主单位群。对这部分作 $r$ 进展开后，$L_s$ 在所需精度内把乘法转成加法，因此

$$
L_s(69^{k_i})\equiv k_iL_s(69)\pmod{r^{s-1}}.
$$

于是

$$
k_i\equiv L_s(K_i)L_s(69)^{-1}\pmod{r^{s-1}}.
$$

而 $s-1=i+1$，恰好覆盖题目给出的 $k_i$ 取值范围，所以同余结果就是原系数。对应代码非常短：

```python
def recover_log(value, s, r):
    modulus = r ** (2 * s + 1)

    def L(x):
        y = pow(x, r ** (s - 1) * (r - 1), modulus)
        return (y - 1) // (r ** s)

    return L(value) * inverse_mod(L(69), r ** (s - 1)) % (r ** (s - 1))

coefficients = [recover_log(value, i + 2, r)
                for i, value in enumerate(public_coefficients)]
```

恢复后可逐项验证 `pow(69, coefficients[i], r**(i+2)) == K[i]`。

### 构造同消息多项式

在多项式环 $(\mathbb Z/n\mathbb Z)[X]$ 中定义

$$
P(X)=\sum_{i=0}^{19}k_iX^i,
$$

$$
f_1(X)=P(X)^{127}-c_1,
$$

$$
f_2(X)=X^{65537}-c_2.
$$

真实消息 $m$ 同时满足 $f_1(m)=f_2(m)=0$。因此二者共享因子 $X-m$。对两个多项式执行欧几里得算法并在每一步将结果首一化：

```sage
R.<X> = PolynomialRing(Zmod(n))
f1 = sum(k * X**i for i, k in enumerate(coefficients))**127 - c1
f2 = X**65537 - c2

def polynomial_gcd(a, b):
    while b:
        a, b = b, a % b
    return a.monic()

g = polynomial_gcd(f1, f2)
assert g.degree() == 1
m = int(-g[0])
flag = long_to_bytes(m)
```

若最大公因式是首一线性多项式 $X-m$，其常数项就是 $-m$。这与 Franklin-Reiter 相关消息攻击的本质相同：同一未知消息经过两个公开且代数相关的多项式映射后，只需在模 $n$ 多项式环中求公共因子，无需分解 $n$。

需要注意，$\mathbb Z/n\mathbb Z$ 不是域，欧几里得过程中不一定总能求首项逆元。若遇到不可逆系数，其与 $n$ 的最大公约数反而可能直接给出因子；本题给定实例能够按官方 solver 的首一化流程正常结束。

## 方法总结

题目表面同时出现离散对数和 RSA，但两个环节都有特殊结构。模 $r^s$ 的单位群允许用 $r$ 进对数型映射把 $69^{k_i}$ 线性化，且公开精度与系数界完全匹配；恢复系数后，两份密文就是关于同一 $m$ 的多项式方程，求多项式最大公因式即可得到 $X-m$。关键是先利用素数幂群结构恢复系数，而不是把 $K_i$ 当成一般大群 DLP。
