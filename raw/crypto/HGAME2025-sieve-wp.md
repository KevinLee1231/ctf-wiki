# sieve

## 题目简述

题目把一个递归函数 `trick(k)` 的结果左移 128 位，再取下一个素数作为 $p$ 和 $q$。由于代码令 `p = q`，RSA 模数为 $n=p^2$。目标是识别 `trick` 中隐藏的素数计数与欧拉函数前缀和，快速计算出 $p$，再恢复私钥解密密文。

## 解题过程

### 1. 化简 `trick` 函数

题目核心代码为：

```python
def trick(k):
    if k > 1:
        mul = prod(range(1, k))
        if k - mul % k - 1 == 0:
            return euler_phi(k) + trick(k - 1) + 1
        return euler_phi(k) + trick(k - 1)
    return 1

e = 65537
p = q = nextprime(trick(e^2 // 6) << 128)
n = p * q
enc = pow(m, e, n)
```

其中 `mul` 是 $(k-1)!$，条件可以改写为

$$
(k-1)!\equiv-1\pmod{k}.
$$

根据 Wilson 定理，对 $k>1$，该条件当且仅当 $k$ 为素数。因此每个 $k$ 都会贡献 $\varphi(k)$，素数还会额外贡献 1。结合 `trick(1)=1=\varphi(1)`，有

$$
\operatorname{trick}(N)
=\sum_{k=1}^{N}\varphi(k)+\pi(N),
\qquad
N=\left\lfloor\frac{65537^2}{6}\right\rfloor
=715849728.
$$

问题由无法直接展开的阶乘递归，转化为两个标准计数函数：素数计数 $\pi(N)$ 和欧拉函数前缀和。

### 2. 计算两个前缀量

`prime_pi(N)` 可以直接计算 $\pi(N)$。欧拉函数和使用杜教筛的递推：由

$$
\sum_{d=1}^{n}\varphi(d)\left\lfloor\frac nd\right\rfloor
=\frac{n(n+1)}2
$$

可得

$$
S(n)=\sum_{i=1}^{n}\varphi(i)
=\frac{n(n+1)}2-
\sum_{l=2}^{n}S\!\left(\left\lfloor\frac nl\right\rfloor\right).
$$

对相同的 $\lfloor n/l\rfloor$ 分块，并预筛前 $10^6$ 项，就不需要为 $7.1\times10^8$ 个整数保存完整数组。

```python
# SageMath
from functools import lru_cache

LIMIT = 10**6


def phi_prefix(limit):
    phi = [0] * (limit + 1)
    phi[1] = 1
    primes = []
    composite = bytearray(limit + 1)

    for i in range(2, limit + 1):
        if not composite[i]:
            primes.append(i)
            phi[i] = i - 1
        for p in primes:
            value = i * p
            if value > limit:
                break
            composite[value] = 1
            if i % p == 0:
                phi[value] = phi[i] * p
                break
            phi[value] = phi[i] * (p - 1)

    prefix = [0] * (limit + 1)
    for i in range(1, limit + 1):
        prefix[i] = prefix[i - 1] + phi[i]
    return prefix


prefix = phi_prefix(LIMIT)


@lru_cache(None)
def summatory_phi(n):
    if n <= LIMIT:
        return prefix[n]

    result = n * (n + 1) // 2
    left = 2
    while left <= n:
        quotient = n // left
        right = n // quotient
        result -= (right - left + 1) * summatory_phi(quotient)
        left = right + 1
    return result


N = 65537**2 // 6
prime_count = ZZ(prime_pi(N))
phi_sum = summatory_phi(N)

print(N)
print(prime_count)
print(phi_sum)
print(prime_count + phi_sum)
```

结果为：

```text
715849728
37030583
155763335410704472
155763335447735055
```

这里需要纠正原 PDF 的一处实质性错误：第 59 页把欧拉函数和写成了 `155763335194435672`，但独立运行杜教筛得到的是 `155763335410704472`；公开选手复现中的 C++ 筛法也记录了后者。错误值无法导出正确的 RSA 参数。

### 3. 构造重复素数 RSA 的私钥

题目给出的密文为：

```python
enc = 2449294097474714136530140099784592732766444481665278038069484466665506153967851063209402336025065476172617376546
```

使用正确的 `trick(N)` 计算 $p$。因为 $n=p^2$，欧拉函数不是 $(p-1)^2$，而是

$$
\varphi(p^2)=p(p-1).
$$

完整解密代码如下：

```python
# SageMath
e = 65537
trick_value = 155763335447735055
enc = 2449294097474714136530140099784592732766444481665278038069484466665506153967851063209402336025065476172617376546

p = ZZ(next_prime(trick_value << 128))
n = p**2
phi_n = p * (p - 1)
d = ZZ(inverse_mod(e, phi_n))
m = power_mod(enc, d, n)
plaintext = int(m).to_bytes((int(m).bit_length() + 7) // 8, "big")

print(p)
print(plaintext)
```

输出为：

```text
53003516465655400667707442798277521907437914663503790163
b'hgame{sieve_is_n0t_that_HArd}'
```

公开题解中保留了题目代码、另一种高内存线性筛实现以及相同的最终明文，可作为交叉核对：[Hgame25 - sieve](https://huangx607087.online/2025/02/21/Hgame25/#5-sieve-7pts-140sol)。

## 方法总结

本题首先用 Wilson 定理把阶乘判定化为素数指示函数，从而得到 $\pi(N)+\sum\varphi(k)$；随后分别用素数计数和杜教筛求值。最后还要注意模数是 $p^2$，必须使用 $\varphi(p^2)=p(p-1)$。与直接分配数 GB 数组相比，预筛加整除分块既更快，也更适合作为可复现题解。
