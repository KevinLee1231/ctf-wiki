# symmetric

## 题目简述

题目生成四个 512 位素数 $p,q,r,s$，公布前三个幂和：

$$
h_1=p+q+r+s,\qquad
h_2=p^2+q^2+r^2+s^2,\qquad
h_3=p^3+q^3+r^3+s^3,
$$

以及 RSA 模数 $N=pqrs$ 和 $c=m^{65537}\bmod N$。四个素数的排列并不重要；泄漏的幂和与乘积都是对称多项式，因此可以先恢复以 $p,q,r,s$ 为根的首一四次多项式，再分解得到所有素因子。

## 解题过程

令 $e_i$ 为四个未知素数的初等对称多项式，$P_i$ 为幂和。题目直接给出 $P_1=h_1$、$P_2=h_2$、$P_3=h_3$ 和 $e_4=N$。由 Newton 恒等式：

$$
\begin{aligned}
e_1&=P_1,\\
2e_2&=e_1P_1-P_2,\\
3e_3&=P_3-e_1P_2+e_2P_1.
\end{aligned}
$$

于是 $p,q,r,s$ 正是下列多项式的四个整数根：

$$
f(x)=x^4-e_1x^3+e_2x^2-e_3x+N.
$$

下面的脚本省略了 `output.txt` 中几个低价值的大整数；实际使用时直接填入即可：

```python
import sympy
from math import prod
from Crypto.Util.number import long_to_bytes

h1 = ...
h2 = ...
h3 = ...
N = ...
ct = ...

e1 = h1
e2 = (h1 * h1 - h2) // 2
e3 = (h3 - e1 * h2 + e2 * h1) // 3

x = sympy.symbols("x")
f = sympy.Poly(
    x**4 - e1*x**3 + e2*x**2 - e3*x + N,
    x,
    domain=sympy.ZZ,
)

primes = []
for factor, multiplicity in sympy.factor_list(f)[1]:
    assert factor.degree() == 1 and multiplicity == 1
    leading, constant = factor.all_coeffs()
    primes.append(int(-constant // leading))

assert len(primes) == 4
assert prod(primes) == N

phi = prod(p - 1 for p in primes)
d = pow(65537, -1, phi)
print(long_to_bytes(pow(ct, d, N)))
```

本地复核中，多项式精确分解出四个不同的 512 位素数。计算 $\varphi(N)=\prod(p_i-1)$ 后恢复私钥指数，得到：

```text
uiuctf{5yMmeTRiC_P0lyS_FoRM_A_B4S15}
```

仓库的 Sage solver 直接把四个幂和/乘积方程组成理想并求 `variety()`，会得到因排列产生的 $4!=24$ 组解。上面的 Newton 恒等式写法消除了排列冗余，也更直接展示了泄漏为什么足以唯一确定根集合。

## 方法总结

- 核心技巧：把已知幂和转换为初等对称多项式，构造根多项式并恢复多素数 RSA 的全部因子。
- 识别信号：题目给出若干未知根的幂和、乘积或其它对称量，而目标只依赖根的无序集合。
- 复用要点：先使用 Newton 恒等式降低变量维度，往往比直接求多元方程组更快、更清楚；恢复因子后仍要验证因子个数、素性和乘积，再计算多素数 RSA 的 $\varphi(N)$。
