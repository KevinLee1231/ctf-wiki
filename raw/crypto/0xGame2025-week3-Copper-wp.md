# Copper!!!

## 题目简述

题目使用标准 RSA-1024，并额外泄露 `gift = p >> 242 << 242`。这等于保留 512 位素因子 $p$ 的高 270 位、清零低 242 位；未知部分小于 $2^{242}$，落在 Coppersmith 单变量小根攻击的范围内。

## 解题过程

设公开高位为 $p_0=\text{gift}$，未知低位为 $x_0$，则：

$$
p=p_0+x_0,\qquad 0\le x_0<2^{242}.
$$

在 $\mathbb Z_n[x]$ 上构造首一多项式：

$$
f(x)=p_0+x.
$$

真正的 $x_0$ 满足 $f(x_0)=p$，因此 $f(x_0)\equiv0\pmod p$，而 $p$ 约为 $n^{1/2}$。对未知因子设置 $\beta=0.5$ 并给出界 $X=2^{242}$，即可恢复小根：

```sage
from sage.all import PolynomialRing, Zmod, ZZ, inverse_mod
from Crypto.Util.number import long_to_bytes

n = ...
c = ...
gift = ...

R.<x> = PolynomialRing(Zmod(n))
roots = (gift + x).small_roots(
    X=2**242,
    beta=0.5,
    epsilon=0.02,
)

p = ZZ(gift + roots[0])
assert n % p == 0
q = n // p

phi = (p - 1) * (q - 1)
d = inverse_mod(65537, phi)
m = pow(c, int(d), n)
print(long_to_bytes(m))
```

分解 $n$ 后就是标准 RSA 解密，仓库参数得到：

```text
0xGame{C0pp3r_4nd_mu1t1pl3_pr0gr3ss1ng!!!}
```

## 方法总结

- `p >> k << k` 是典型的素因子高位泄露，未知量是低 $k$ 位小根。
- 构造 $f(x)=p_0+x$ 后，根不是模 $n$ 为零，而是模未知大因子 $p$ 为零，因此要给 `small_roots()` 设置与因子规模对应的 $\beta$。
- 恢复候选后必须验证 `n % p == 0`，再进入常规 RSA 私钥计算。
