# ezRSA

## 题目简述

题目使用三素数 RSA：

$$
n=pqr,\qquad e=65537.
$$

其中 $s$ 是 300 位素数，服务给出的第一项不是 $s$ 本身，而是

$$
F(s)=160s^5-4999s^4+3s^3+1.
$$

另外还给出

$$
k=(p-s)d
$$

以及

$$
\mathit{gift}
=p^{(q-1)(r-1)qr}(qr+1)^q
\pmod{(qr)^3}.
$$

需要依次恢复 $s$、利用 $k$ 分离出 $p$，再从 `gift` 中区分 $q$ 与 $r$。

## 解题过程

### 1. 二分恢复 $s$

`getPrime(300)` 保证

$$
2^{299}\le s<2^{300}.
$$

在这个区间内 $F(s)$ 严格递增，因此可以对整数做二分，并用等式验证结果。

### 2. 利用指数同余求出 $p$

设

$$
ed=1+t\varphi(n).
$$

由 $k=(p-s)d$ 可得

$$
ek=(p-s)\bigl(1+t\varphi(n)\bigr).
$$

因为 $p-1\mid\varphi(n)$，所以模 $p-1$ 有

$$
ek\equiv p-s\equiv1-s\pmod{p-1}.
$$

于是对与 $n$ 互素的底数 $g$，

$$
g^{ek}-g^{1-s}\equiv0\pmod p.
$$

该差值通常不同时被 $q$、$r$ 整除，因此

$$
p=\gcd\left(g^{ek}-g^{1-s},n\right).
$$

原稿把两个指数写成在模 $n$ 下直接相等，这是不准确的；真正成立的是上面的模 $p-1$ 指数同余。

### 3. 改变模数拆出 $q$

令 $N=qr$。指数

$$
(q-1)(r-1)qr=\varphi(N)N=\varphi(N^2),
$$

所以由欧拉定理，

$$
p^{(q-1)(r-1)qr}\equiv1\pmod{N^2}.
$$

再对 $(N+1)^q$ 使用二项式展开：

$$
(N+1)^q\equiv1+qN\pmod{N^2}.
$$

因此

$$
\mathit{gift}\bmod N^2=1+qN,
$$

直接得到

$$
q=\frac{(\mathit{gift}\bmod N^2)-1}{N},
\qquad r=\frac Nq.
$$

完整求解骨架如下，五个大整数用题目输出替换：

```python
from Crypto.Util.number import GCD, inverse, long_to_bytes

E = 0x10001


def polynomial(x):
    return 160 * x**5 - 4999 * x**4 + 3 * x**3 + 1


def recover_s(value):
    left = 1 << 299
    right = (1 << 300) - 1

    while left <= right:
        middle = (left + right) // 2
        current = polynomial(middle)
        if current < value:
            left = middle + 1
        elif current > value:
            right = middle - 1
        else:
            return middle

    raise ValueError("F(s) 与 300 位整数区间不匹配")


S_VALUE = ...
N_RSA = ...
K_VALUE = ...
GIFT = ...
CIPHER = ...

s = recover_s(S_VALUE)

# 3^(1-s) mod n 写成 (3^-1)^(s-1)，避免负指数兼容性问题。
base_inverse = inverse(3, N_RSA)
difference = (
    pow(3, E * K_VALUE, N_RSA)
    - pow(base_inverse, s - 1, N_RSA)
) % N_RSA

p = GCD(difference, N_RSA)
assert 1 < p < N_RSA

qr = N_RSA // p
reduced_gift = GIFT % (qr**2)
assert (reduced_gift - 1) % qr == 0

q = (reduced_gift - 1) // qr
assert 1 < q < qr and qr % q == 0
r = qr // q
assert p * q * r == N_RSA

phi = (p - 1) * (q - 1) * (r - 1)
d = inverse(E, phi)
print(long_to_bytes(pow(CIPHER, d, N_RSA)))
```

输出为：

```text
moectf{5om3_M4th_m4y,b3_c0oool!}
```

## 方法总结

- 核心技巧：单调多项式逆像、由 $ed\equiv1\pmod{\varphi(n)}$ 构造 GCD 因子，以及通过降低 `gift` 的模数保留一阶二项式项。
- 识别信号：RSA 辅助量同时出现 $(p-s)d$、多素数模数和 $(qr+1)^q$ 时，应分别检查指数同余与模 $N^2$ 的二项式展开。
- 复用要点：推导中必须写清同余发生在哪个模数下；恢复每个因子后都要验证整除关系和 $pqr=n$。
