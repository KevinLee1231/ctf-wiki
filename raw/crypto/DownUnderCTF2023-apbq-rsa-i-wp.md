# DownUnderCTF 2023 apbq rsa i Writeup

## 题目简述

题目使用标准 RSA，公开 $n=pq$、密文 $c=m^e\bmod n$，以及两个线性提示：

$$
h_i=a_i p+b_i q.
$$

其中 $p,q$ 为 1024 位素数，$a_i<2^{12}$，而 $b_i<2^{312}$。关键是穷举很小的 $a_i$，消去 $p$ 后用最大公因数分解 $n$。

## 解题过程

设两个提示为：

$$
h_0=a_0p+b_0q,\qquad h_1=a_1p+b_1q.
$$

交叉相乘后：

$$
a_1h_0-a_0h_1=(a_1b_0-a_0b_1)q.
$$

当枚举值命中真实的 $(a_0,a_1)$ 时，右侧是 $q$ 的倍数，因此与 $n$ 求最大公因数即可得到非平凡因子。完整求解逻辑为：

```python
from itertools import product
from math import gcd
from Crypto.Util.number import long_to_bytes

# output.txt 定义 n、c、hints
exec(open("output.txt", "r").read())
e = 0x10001

factor = None
for left, right in product(range(2**12), repeat=2):
    candidate = gcd(left * hints[0] - right * hints[1], n)
    if 1 < candidate < n:
        factor = candidate
        break

assert factor is not None
q = factor
p = n // q
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)
print(long_to_bytes(pow(c, d, n)).decode())
```

解密得到：

```text
DUCTF{gcd_1s_a_g00d_alg0r1thm_f0r_th3_t00lbox}
```

## 方法总结

线性泄漏中，只要某一组系数足够小，就可以穷举系数并构造消元式。消元结果不必等于素因子本身，只要是其中一个素因子的倍数且不同时含另一个素因子，`gcd(..., n)` 就会把 RSA 模数分解。
