# GlacierCTF2023 - ARISAI

## 题目简述

题目使用 $e=65537$ 的多素数 RSA：模数 $N$ 是 256 个 24 位素数的乘积，密文为 $c=m^e\bmod N$。$N$ 虽然很大，但每个素因子最多只有 $2^{24}$，而且生成数据时还有三个素因子被重复选中。安全性取决于最小因子的强度，而不是模数十进制长度。

## 解题过程

直接对 $2\leq i<2^{24}$ 试除即可完整分解 $N$。记录每个因子的指数 $k_i$，不能把重复因子当成互异素数；若

$$
N=\prod_i p_i^{k_i},
$$

则

$$
\varphi(N)=\prod_i (p_i-1)p_i^{k_i-1}.
$$

附件中重复出现的三个因子是 `8725153`、`11369903` 和 `16177433`。得到完整分解后计算 $d=e^{-1}\bmod\varphi(N)$，再求 $m=c^d\bmod N$：

```python
from collections import Counter
from Crypto.Util.number import long_to_bytes

limit = 1 << 24
work = N
factors = []

for p in range(2, limit):
    while work % p == 0:
        factors.append(p)
        work //= p
    if work == 1:
        break

assert work == 1
phi = 1
for p, k in Counter(factors).items():
    phi *= (p - 1) * p ** (k - 1)

d = pow(e, -1, phi)
print(long_to_bytes(pow(ct, d, N)))
```

输出为：

```text
gctf{maybe_I_should_have_used_bigger_primes}
```

## 方法总结

多素数 RSA 并不会自动更安全。看到异常巨大的 $N$ 时，应同时检查因子数量和单个因子的位数；若因子会重复，还必须按 $p^k$ 的公式计算欧拉函数，否则私钥指数会错误。
