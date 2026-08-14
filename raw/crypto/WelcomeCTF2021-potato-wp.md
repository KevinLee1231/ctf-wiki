# Potato

## 题目简述

WelcomeCTF2021 的 Potato 在有限域 $GF(2^{272})$ 中工作。flag 被编码为域元素 $g$ 的多项式系数，题目给出指数 $a$、不可约多项式定义的域，以及 $g^a$ 的系数，要求恢复 $g$。

## 解题过程

$GF(2^{272})$ 的非零元素构成阶为 $2^{272}-1$ 的乘法群。只要

$$
\gcd(a,2^{272}-1)=1,
$$

指数 $a$ 就在该群阶下存在逆元

$$
d=a^{-1}\pmod{2^{272}-1}.
$$

于是

$$
(g^a)^d=g^{ad}=g.
$$

官方 Sage 脚本直接构造有限域，并按题目给出的系数重建 $g^a$：

```python
from Crypto.Util.number import long_to_bytes

n = 272
R.<z> = PolynomialRing(GF(2))
modulus = z^272 + z^9 + z^3 + z^2 + 1
F.<x> = GF(2^n, modulus=modulus)

encoded = F(0)
for i, bit in enumerate(coeff):
    if bit:
        encoded += x^i

order = F.order() - 1
d = inverse_mod(a, order)
g = encoded^d
```

域元素的多项式系数按题目约定由低次项到高次项排列。将系数连接为二进制串，再转为整数和字节即可得到 flag：

```python
bits = "".join(map(str, g.polynomial().list()))
print(long_to_bytes(int(bits, 2)))
```

官方脚本输出：

```text
greyhats{P0lyn0M1al5_Ar3_SuP3RI0R}
```

这里不能把 $g^a$ 当作普通整数直接开 $a$ 次根；所有乘法、幂运算和约简都必须在题目给出的不可约多项式所定义的 $GF(2^{272})$ 中进行。

## 方法总结

本题与 RSA 的“求逆指数”结构相似，但群阶从 $\varphi(N)$ 换成了有限域非零乘法群的阶 $2^n-1$。解题重点是确认表示方式与群阶，随后用 Sage 在正确的域中求逆指数。最后的系数位序若反转，即使域运算正确也会得到错误字节串。
