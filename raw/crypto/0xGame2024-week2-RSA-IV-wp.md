# RSA-IV

## 题目简述

服务随机生成一个 96 位明文 `m`，依次提供四组带不同弱点的 RSA 参数。四题分别对应低指数直接开根、泄露 `dp`、过小私钥和共模攻击；每组提交正确的 `m` 后累计一分，四分时返回 flag。

## 解题过程

### challenge 0：低指数且未发生模约减

参数使用 300 位左右的模数和 `e = 3`，而 $m$ 只有 96 位，因此通常满足：

$$
m^3<N
$$

于是 $c=m^3$ 是普通整数等式，直接对 $c$ 开整数立方根。

### challenge 1：泄露 $d_p$

已知：

$$
ed_p\equiv1\pmod{p-1}
$$

所以存在 $1\le k<e$ 使：

$$
ed_p-1=k(p-1)
$$

遍历 $k$，若 $k\mid(ed_p-1)$，则候选：

$$
p=\frac{ed_p-1}{k}+1
$$

再检查 $p\mid N$ 即可分解模数并正常解密。

### challenge 2：21 位私钥

源码用 `getPrime(21)` 生成 $d$，范围只有 $[2^{20},2^{21})$。遍历该区间内的质数，计算 $m=c^d\bmod N$；正确明文不超过 96 位，且重新加密应得到原密文。该范围足够小，无需引入 LLL 或 Boneh–Durfee。

### challenge 3：共模攻击

同一明文在相同模数 $N$ 下使用互质指数 $e_1,e_2$：

$$
c_1\equiv m^{e_1}\pmod N,\qquad c_2\equiv m^{e_2}\pmod N
$$

扩展欧几里得算法求出 $a,b$，满足 $ae_1+be_2=1$，于是：

$$
m\equiv c_1^a c_2^b\pmod N
$$

负指数通过模逆处理。

四个求解函数如下：

```python
from gmpy2 import iroot
from sympy import primerange


def egcd(a, b):
    if b == 0:
        return a, 1, 0
    g, x, y = egcd(b, a % b)
    return g, y, x - (a // b) * y


def signed_pow(value, exponent, modulus):
    if exponent >= 0:
        return pow(value, exponent, modulus)
    return pow(pow(value, -1, modulus), -exponent, modulus)


def solve0(N, e, c):
    m, exact = iroot(c, 3)
    assert exact
    return int(m)


def solve1(N, e, c, dp):
    value = e * dp - 1
    for k in range(1, e):
        if value % k:
            continue
        p = value // k + 1
        if N % p == 0:
            q = N // p
            phi = (p - 1) * (q - 1)
            d = pow(e, -1, phi)
            return pow(c, d, N)
    raise ValueError("未找到 p")


def solve2(N, e, c):
    for d in primerange(1 << 20, 1 << 21):
        m = pow(c, d, N)
        if m.bit_length() <= 96 and pow(m, e, N) == c:
            return m
    raise ValueError("未找到 d")


def solve3(N, e1, c1, e2, c2):
    g, a, b = egcd(e1, e2)
    assert g == 1
    return signed_pow(c1, a, N) * signed_pow(c2, b, N) % N
```

连接当前实例后，分别选择 `0` 到 `3`，把服务输出的元组传给对应函数，再提交返回的十进制整数。四题全部通过后服务输出：

```text
0xGame{2b5e024a-3c62-4f4a-afe0-b81851d9efc8}
```

## 方法总结

四个弱点都来自 RSA 使用条件被破坏：明文太小、CRT 私钥分量泄露、私钥空间过小、同模数重复使用。分析时先比较参数位数和已泄露关系，通常能选择比通用格攻击更简单、可验证的解法。
