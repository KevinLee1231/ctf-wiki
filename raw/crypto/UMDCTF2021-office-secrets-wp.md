# Office Secrets

## 题目简述

题目给出两组 RSA 公钥参数和密文。两次加密复用了相同的模数 $n$ 和明文 $m$，但使用互素的指数 $e_1=65537$、$e_2=65541$，因此可使用 RSA 共模攻击。

## 解题过程

两组密文满足：

$$
c_1\equiv m^{e_1}\pmod n,\qquad
c_2\equiv m^{e_2}\pmod n.
$$

由于 $\gcd(e_1,e_2)=1$，扩展欧几里得算法可以求出整数 $a,b$，使：

$$
a e_1+b e_2=1.
$$

于是：

$$
m\equiv c_1^a c_2^b\pmod n.
$$

指数为负时先对相应密文求模逆：

```python
from Crypto.Util.number import long_to_bytes
from sympy import gcdex

a, b, g = map(int, gcdex(e1, e2))

def signed_pow(c, e, n):
    if e < 0:
        return pow(pow(c, -1, n), -e, n)
    return pow(c, e, n)

m = signed_pow(c1, a, n) * signed_pow(c2, b, n) % n
print(long_to_bytes(m))
```

恢复结果为：

```text
UMDCTF-{Sh4r1ng_I5_d3f1n1t3ly_n0t_c4r!ng}
```

## 方法总结

RSA 不仅要避免复用素数，也不能在相同模数下用互素指数加密同一明文。共模攻击不需要分解 $n$，只利用 Bézout 系数把两个指数线性组合成 1。实现时最容易忽略负指数对应的模逆运算。
