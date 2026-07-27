# myRSA

## 题目简述

题目生成 RSA 素数时使用了 LFSR。该 LFSR 的周期只有 2697，却连续输出了 4096 bit，因此后面的

$$
4096-2697=1399
$$

bit 必然与前面重复。对题目中的两个 RSA 模数 $N_1=p_1q_1$、$N_2=p_2q_2$ 分析后，可确认两个素数在不同偏移处共享 877 个连续 bit：

```python
bin(q2)[2:][256:256 + 877] == bin(p2)[2:][905:905 + 877]
```

普通 implicit factorization 通常假设两个素数在相同位置共享高位、低位或中间位；本题共享片段的位置不同，因此使用 [Generalized Implicit Factorization Problem](https://arxiv.org/abs/2304.08718)（GIFP）中的多元格攻击。

该论文把“共享连续 bit 位于不同、甚至未知位置”的 RSA 分解问题写成小根方程，通过 Coppersmith 格构造、LLL 和 Gröbner basis 恢复未知片段。论文给出的 [官方 Sage 实现](https://github.com/fffmath/gifp) 还说明：引入新变量 $w$ 并加入约束 $zw-N_2=0$ 后，理论界可写为

$$
\gamma>4\alpha(1-\sqrt{\alpha});
$$

实际恢复因子时不一定需要 Gröbner basis 满足理想化的“固定数量、代数独立”形态。

## 解题过程

### 参数映射

本题参数对应：

$$
\begin{aligned}
n&=2048, & \alpha n&=256,\\
\gamma n&=877, & \beta_1n&=10,\\
\beta_2n&=659.
\end{aligned}
$$

其中 $\gamma n$ 是重复的共享片段长度，$\beta_1n,\beta_2n$ 描述该片段在两个素数中的不同偏移。取格参数 $m=6$，并设置变量界：

```python
modulus_bit_length = 2048
alpha_bit_length = 256
share_bit_length = 877
beta1_bit_length = 10
beta2_bit_length = 659

X = 2**beta2_bit_length
Y = 2**(
    modulus_bit_length
    - alpha_bit_length
    - share_bit_length
    - beta1_bit_length
)
Z = 2**alpha_bit_length
W = 2**(modulus_bit_length - alpha_bit_length)
M = 2**(beta2_bit_length - beta1_bit_length)
```

交换 $N_1,N_2$ 使符号满足脚本的 $p>q$ 假设。建立

$$
f(x,y,z,w)
=xz+2^{\beta_2+\gamma}yz+N_2
$$

并在商环

$$
\mathbb Z[x,y,z,w]/\langle zw-N_2\rangle
$$

中生成 Coppersmith shifts。每个多项式按 $(X,Y,Z,W)$ 缩放后组成格基，执行 LLL，再还原成有理系数多项式：

```sage
R.<x, y, z, w> = PolynomialRing(ZZ, 4, order="lex")
f = x*z + 2**(beta2_bit_length + share_bit_length)*y*z + N2
quotient = R.quotient(z*w - N2)

m = 6
alpha = QQ(alpha_bit_length) / modulus_bit_length
t = round((1 - sqrt(alpha)) * m)
s = round(sqrt(alpha) * m)

def eliminate_N2(polynomial, modulus_value):
    result = R.zero()
    for monomial in polynomial.monomials():
        coefficient = polynomial.monomial_coefficient(monomial)
        reduced = coefficient % modulus_value
        if reduced == 0:
            reduced = modulus_value
        result += monomial * reduced
    return result

shifts = []
modulus = M**m * N1**t
N2_inverse = inverse_mod(N2, modulus)

for i in range(m + 1):
    for j in range(m - i + 1):
        g = (
            (y*z)**j
            * w**s
            * f**i
            * M**(m - i)
            * N1**max(t - i, 0)
            * N2_inverse**min(i + j, s)
        )
        shifts.append(eliminate_N2(quotient(g).lift(), modulus))

lattice, monomials = Sequence(shifts).coefficient_matrix()
bounds = [X, Y, Z, W]
factors = [monomial(*bounds) for monomial in monomials]
for column, factor in enumerate(factors):
    lattice.rescale_col(column, factor)

lattice = lattice.LLL().change_ring(QQ)
for column, factor in enumerate(factors):
    lattice.rescale_col(column, 1 / factor)

polynomials = Sequence(filter(None, lattice * vector(monomials)))
polynomials.insert(0, w*z - N2)
G = ideal(polynomials[:20]).groebner_basis()
```

### 不要求 Gröbner basis 恰好有四个多项式

直接套用参考实现时容易误以为必须得到四个形状完美的多项式。官方仓库的说明强调，这不是恢复因子的必要条件。引入 $w$ 后，只要若干基多项式之间存在公共线性因子或可读出的比例关系，就可能得到

$$
z_0w_0=N_2
$$

以及关于 $x_0,y_0,w_0$ 的线性关系。

在本题输出中，前两个较简单的基多项式已足以确定 $x,y,w$ 的比例。又因为 $w_0$ 本身对应 $N_2$ 的素因子 $p_2$，可检查 Gröbner basis 系数的分母与 $N_2$ 的最大公因数：

```sage
def nontrivial_factor(denominator):
    if denominator == 1:
        return None
    factor = gcd(denominator, N2)
    if factor in (1, N2):
        return None
    return factor

p2 = None
for polynomial in G:
    for coefficient, _ in polynomial:
        p2 = nontrivial_factor(coefficient.denominator())
        if p2:
            break
    if p2:
        break

q2 = N2 // p2
```

把 $w=p_2$ 代回两个简单多项式求出 $x_0,y_0$，再利用共享片段的位移关系分解 $N_1$：

```sage
x0 = int(G[0](w=p2).univariate_polynomial().roots()[0][0])
y0 = int(G[1](w=p2).univariate_polynomial().roots()[0][0])

p1 = gcd(
    N1,
    p2 + x0 + y0 * 2**(share_bit_length + beta2_bit_length),
)
q1 = N1 // p1
```

最后按普通 RSA 解密：

```sage
from Crypto.Util.number import long_to_bytes

phi = (p1 - 1) * (q1 - 1)
d = inverse_mod(e, phi)
flag = long_to_bytes(pow(c, d, N1))
```

原截图只展示了官方仓库 README 中的 Gröbner basis 示例，属于可可靠转写的纯文本内容，因此没有保留图片。论文和代码 URL 仍保留作完整推导与实现来源，但本题所需的周期重叠、GIFP 参数、格构造和非标准 Gröbner basis 读取方式均已写入正文。

## 方法总结

- 核心技巧：由 LFSR 周期确定两个 RSA 素数在不同偏移处共享长 bit 段，再使用 GIFP 的多元 Coppersmith/LLL 攻击分解模数。
- 识别信号：随机源输出长度超过周期、两个模数的素数片段出现错位重叠时，不能只套用“同位置共享高低位”的 IFP，应建立带偏移变量的广义模型。
- 复用要点：Gröbner basis 的形状不必与论文示例完全一致；优先检查多项式公因子、系数比例，以及系数分母和模数的非平凡 GCD，这些信息可能直接泄露素因子。
