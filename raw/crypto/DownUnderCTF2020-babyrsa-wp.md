# DownUnderCTF 2020 - babyrsa

## 题目简述

题目使用标准 RSA 参数 $n=pq$、$e=65537$ 加密 flag，同时泄露：

$$
s=(557p-127q)^{n-p-q}\bmod n.
$$

直接分解 2048 位模数不可行，但泄漏指数恰好是 $\varphi(n)-1$，可以把 $s$ 变成关于 $p,q$ 的线性关系，再构造可在整数环中直接分解的二次多项式。

## 解题过程

由于

$$
\varphi(n)=(p-1)(q-1)=n-p-q+1,
$$

令 $A=557p-127q$，则题目给出的指数满足 $n-p-q=\varphi(n)-1$。在 $A$ 与 $n$ 互素时，由 Euler 定理得到：

$$
s=A^{\varphi(n)-1}\equiv A^{-1}\pmod n.
$$

所以

$$
S=s^{-1}\bmod n=557p-127q.
$$

接着构造：

$$
f(x)=127\cdot557x^2+Sx-n.
$$

它在整数环中的真实分解为：

$$
f(x)=(127x+p)(557x-q).
$$

因此对 $f(x)$ 做符号分解即可从两个一次因子的常数项读出 $p$ 和 $q$。恢复因子后按普通 RSA 计算私钥指数并解密：

```sage
from Crypto.Util.number import long_to_bytes

# n、s、c 取自题目 output.txt
S = inverse_mod(s, n)

R.<x> = ZZ[]
f = 127 * 557 * x^2 + S * x - n
factors = f.factor()

# 也可从两个一次因子的常数项读取；最后用 n % p == 0 验证。
candidates = []
for factor, multiplicity in factors:
    for coeff in factor.list():
        value = abs(ZZ(coeff))
        if value > 1 and n % value == 0:
            candidates.append(value)

p = candidates[0]
q = n // p
assert p * q == n

e = 0x10001
d = inverse_mod(e, (p - 1) * (q - 1))
m = power_mod(c, d, n)
print(long_to_bytes(int(m)).decode())
```

解密结果为：

```text
DUCTF{e4sy_RSA_ch4ll_t0_g3t_st4rt3d}
```

## 方法总结

RSA 题中的辅助幂值要先和 $\varphi(n)$ 对齐。本题的核心不是暴力分解 $n$，而是识别 $\varphi(n)-1$ 会把底数变成模逆，从而泄露 $557p-127q$。当同时已知 $pq$ 和 $ap+bq$ 一类关系时，构造可分解的二次多项式通常能直接恢复两个素因子。
