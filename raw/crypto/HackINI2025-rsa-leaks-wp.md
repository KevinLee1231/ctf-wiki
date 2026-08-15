# rsa_leaks

## 题目简述

题目构造三素数 RSA 模数 $n=pqr$，其中 $p,q$ 为 512 位素数、$r$ 为 2048 位素数。同时泄露一个线性同余生成器的隔项状态：原始递推为

$$
x_{t+1}\equiv px_t+q\pmod r,
$$

输出数组只保留 $x_0,x_2,x_4,\ldots$。由于 LCG 的乘数、增量和模数恰好就是 RSA 因子或其组合，恢复 LCG 参数即可分解 $n$。

## 解题过程

### 把隔项输出写成新的 LCG

连续执行两次原递推：

$$
x_{t+2}\equiv p(px_t+q)+q
\equiv p^2x_t+q(p+1)\pmod r.
$$

令泄露序列 $X_j=x_{2j}$，则

$$
X_{j+1}\equiv AX_j+C\pmod r,qquad
A=p^2,\quad C=q(p+1).
$$

### 从状态差恢复模数

定义整数差 $\Delta_j=X_{j+1}-X_j$。模 $r$ 有

$$
\Delta_{j+1}\equiv A\Delta_j\pmod r.
$$

因此每个量

$$
T_j=\Delta_{j+2}\Delta_j-\Delta_{j+1}^2
$$

都是 $r$ 的倍数。对多组 $|T_j|$ 求最大公约数即可恢复 $r$；若结果带有额外小因子，还应结合 `n % r == 0` 做筛选。

随后由前三个状态计算

$$
A\equiv(X_2-X_1)(X_1-X_0)^{-1}\pmod r.
$$

本题的 $A$ 是整数平方 $p^2$，对其开精确平方根得到 $p$，最后由 $q=n/(pr)$ 恢复剩余因子：

```python
from functools import reduce
from math import gcd
from gmpy2 import iroot
from Crypto.Util.number import inverse, long_to_bytes

def recover_modulus(sequence):
    multiples = []
    for i in range(len(sequence) - 3):
        d1 = sequence[i + 1] - sequence[i]
        d2 = sequence[i + 2] - sequence[i + 1]
        d3 = sequence[i + 3] - sequence[i + 2]
        multiples.append(abs(d3 * d1 - d2 * d2))
    return reduce(gcd, multiples)

n, e, c = ..., 65537, ...
X = [...]  # 题目泄露的六个隔项状态

r = recover_modulus(X)
assert n % r == 0

A = ((X[2] - X[1]) * inverse((X[1] - X[0]) % r, r)) % r
p, exact = iroot(A, 2)
assert exact
p = int(p)
q = n // (p * r)
assert p * q * r == n

phi = (p - 1) * (q - 1) * (r - 1)
d = pow(e, -1, phi)
m = pow(c, d, n)
assert pow(m, e, n) == c
print(long_to_bytes(m))
```

运行仓库官方 solver 得到：

```text
shellmates{N0_w4y_tH3Y_g3n3r4tEd_r4Nd0m_NUMBERs_Th1S_w4Y_iN_tHe_p4$T}
```

## 方法总结

- 核心技巧：利用 LCG 状态差的行列式型不变量恢复模数，再从隔项递推的乘数 $A=p^2$ 取平方根获得 RSA 因子。
- 识别信号：RSA 因子被复用为 PRNG 的乘数、增量或模数，且泄露至少四个等间隔状态时，应先推导采样后递推，而不是直接套连续状态公式。
- 复用要点：隔 $k$ 项采样会把乘数变为 $a^k$；GCD 结果和开根结果都必须用 `n` 的整除关系验证。
