# rsa1

## 题目简述

题目使用标准 RSA 参数 $n=pq$、$e=65537$ 加密 Flag，同时泄露：

$$
t_1=(p+q)^e\bmod n,\qquad t_2=(p-q)^e\bmod n.
$$

因为 $e$ 为奇数，这两项在模 $p$、模 $q$ 下存在可直接利用的符号关系，可以通过最大公因数分解 $n$。

## 解题过程

在模 $p$ 下有 $p+q\equiv q$、$p-q\equiv-q$，所以：

$$
t_1+t_2\equiv q^e+(-q)^e\equiv0\pmod p.
$$

在模 $q$ 下则有 $p+q\equiv p$、$p-q\equiv p$，因此：

$$
t_1-t_2\equiv0\pmod q.
$$

由此恢复两个素因子，再按标准 RSA 流程求私钥并解密：

```python
from math import gcd
from Crypto.Util.number import long_to_bytes

p = gcd(t1 + t2, n)
q = gcd(t1 - t2, n)
assert p * q == n

phi = (p - 1) * (q - 1)
d = pow(65537, -1, phi)
print(long_to_bytes(pow(c, d, n)))
```

输出为：

```text
greyhats{algebraic_manipulation_is_fun_too_FEiLZM5oNPMXu87qXpsX}
```

## 方法总结

RSA 中任何关于 $p,q$ 的附加代数值都应分别模 $p$、模 $q$ 化简。这里无需展开高次幂；奇数指数保证负号保留，使和与差分别含有 $p$、$q$ 因子。实际求解后必须检查 $pq=n$，以排除最大公因数为 $1$ 或 $n$ 的退化结果。
