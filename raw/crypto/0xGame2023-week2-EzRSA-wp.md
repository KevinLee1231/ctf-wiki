# EzRSA

## 题目简述

服务包含三个独立 RSA 分解场景：费马小定理泄露因子、$p-1$ 光滑导致 Pollard $p-1$ 攻击，以及两个模数的大素因子极其接近而使小因子之比出现在连分数渐进分数中。每关取得因子后按标准 RSA 解密，并把明文提交给检查接口。

## 解题过程

### Challenge 1：gift 泄露因子

源码给出 $n=pq$ 和：

$$
gift=g^{r(p-1)}\bmod n.
$$

由费马小定理，$gift\equiv1\pmod p$，所以 $p$ 整除 `gift-1`：

```python
from Crypto.Util.number import GCD

p = GCD(gift - 1, n)
assert 1 < p < n
q = n // p
```

### Challenge 2：Pollard p-1

`GetMyPrime()` 先把许多小素数相乘，再加 1 并检查是否为素数，因此 $p-1$、$q-1$ 都由小素数组成。令指数逐步包含更多小因子，若 $a^M\equiv1\pmod p$，则 `gcd(a^M-1,n)` 会暴露 $p$：

```python
from math import gcd

a = 2
for bound in range(2, 100000):
    a = pow(a, bound, n)  # 累积指数包含 bound! 的因子
    p = gcd(a - 1, n)
    if 1 < p < n:
        q = n // p
        break
else:
    raise RuntimeError("increase the smoothness bound")
```

### Challenge 3：连分数恢复小因子

两个模数分别为 $n_1=q_1p_1$、$n_2=q_2p_2$，其中 $q_i$ 只有 128 位，而 896 位的 $p_2$ 是 $p_1$ 的下一个素数，二者非常接近。因此：

$$
\frac{n_1}{n_2}=\frac{q_1p_1}{q_2p_2}\approx\frac{q_1}{q_2}.
$$

枚举 $n_1/n_2$ 的连分数渐进分数，直到分子、分母分别整除两个模数：

```python
def convergents(num, den):
    cf = []
    while den:
        cf.append(num // den)
        num, den = den, num % den

    p0, p1, q0, q1 = 0, 1, 1, 0
    for a in cf:
        p0, p1 = p1, a * p1 + p0
        q0, q1 = q1, a * q1 + q0
        yield p1, q1

for q_small_1, q_small_2 in convergents(n1, n2):
    if (
        q_small_1 > 1 and q_small_2 > 1
        and n1 % q_small_1 == 0
        and n2 % q_small_2 == 0
    ):
        p_large_1 = n1 // q_small_1
        p_large_2 = n2 // q_small_2
        break
```

三关都用同一标准解密函数；第三关的密文属于 `n2`：

```python
from Crypto.Util.number import inverse, long_to_bytes

def decrypt_rsa(n, e, c, p, q):
    assert p * q == n
    d = inverse(e, (p - 1) * (q - 1))
    return long_to_bytes(pow(c, d, n))
```

## 方法总结

RSA 分解题应从素数生成方式和额外公开量入手，而不是只看模数位数。三关分别破坏了“公开量不泄露因子”“$p-1$ 含大素因子”和“不同模数的因子无可利用关系”这些前提；得到因子后的解密流程完全相同。
