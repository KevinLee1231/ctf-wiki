# MiniLCTF2022 factorchal Writeup

## 题目简述

服务生成 RSA 模数 $n=pq$，但私钥指数被故意设置为

$$
d=k p,
$$

其中 $k$ 是小于 $2^{27}$ 的素数。题目看似要求直接分解大模数，真正的弱点却是私钥指数含有素因子 $p$，且小素数 $k$ 可枚举。

## 解题过程

RSA 满足 $ed\equiv1\pmod{\varphi(n)}$，所以存在整数 $u$ 使

$$
e k p=u\varphi(n)+1.
$$

任选与 $n$ 互素的底数 $r$。模 $p$ 考察上式，由费马小定理可得

$$
r^{ekp}\equiv r\pmod p,
$$

而 $r^{ekp}=(r^{ek})^p\equiv r^{ek}\pmod p$，因此

$$
r^{ek}-r\equiv0\pmod p.
$$

依次枚举 $2^{27}$ 范围内的素数 $k$，计算

$$
g=\gcd(r^{ek}-r,n).
$$

当 $1<g<n$ 时便得到 $p=g$，继而求出 $q=n/p$、$\varphi(n)$ 和常规私钥指数 $e^{-1}\bmod\varphi(n)$，最后解密密文。实现时应使用模幂，不能先计算完整的 $r^{ek}$：

```python
from gmpy2 import gcd, powmod
from sympy import primerange

for k in primerange(2, 1 << 27):
    value = (powmod(r, e * k, n) - r) % n
    p = int(gcd(value, n))
    if 1 < p < n:
        q = n // p
        phi = (p - 1) * (q - 1)
        d = pow(e, -1, phi)
        message = pow(c, d, n)
        break
```

用附件参数复现得到：

```text
miniLCTF{e1ba4ea5-fafc-4ebf-be83-c4b2c6b9d918}
```

## 方法总结

RSA 中只要 $d$ 与某个素因子存在特殊代数关系，就可能把“搜索小参数”转化为“构造一个必被该素因子整除的数”。本题的关键等式是 $p\mid r^{ek}-r$；`gcd` 将这一模 $p$ 的关系提升为对 $n$ 的因子恢复。实际实现还应更换几个随机底数，以规避偶然同时被 $p$、$q$ 整除而得到 $n$ 的退化情况。
