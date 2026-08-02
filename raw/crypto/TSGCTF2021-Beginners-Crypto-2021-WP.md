# TSGCTF2021 Beginner's Crypto 2021 WP

## 题目简述

题目生成两个 1024 位强素数 $p,q$，公开它们以及同一 flag 在同一 RSA 模数 $N=pq$ 下的三份密文。三个指数由秘密素数 $e$ 构造：

```python
assert isPrime(e)
assert isPrime(e + 2)
assert isPrime(e + 4)

e1 = pow(e,     0x10001, phi)
e2 = pow(e + 2, 0x10001, phi)
e3 = pow(e + 4, 0x10001, phi)

c1 = pow(flag, e1, N)
c2 = pow(flag, e2, N)
c3 = pow(flag, e3, N)
```

目标是先从三个素数约束确定 $e$，再利用同一明文、同一模数和互素指数恢复 flag。

## 解题过程

在 $e,e+2,e+4$ 三个数中，必有一个能被 3 整除。三者又都必须是素数，因此这个数只能等于 3。由于 $e$ 本身也是素数，唯一可能是：

$$
e=3,\qquad e+2=5,\qquad e+4=7.
$$

虽然程序先把指数模 $\varphi(N)$ 化简，但 RSA 幂运算满足：

$$
m^{a\bmod\varphi(N)}\equiv m^a\pmod N.
$$

所以前两份密文可以视为：

$$
c_1=m^{3^{65537}}\bmod N,
\qquad
c_2=m^{5^{65537}}\bmod N.
$$

两个整数指数互素。用扩展欧几里得算法求 $x,y$，使：

$$
x\cdot3^{65537}+y\cdot5^{65537}=1.
$$

于是：

$$
c_1^x c_2^y
\equiv m^{x3^{65537}+y5^{65537}}
\equiv m\pmod N.
$$

负指数由模逆处理。官方 solver 的核心代码为：

```python
N = p * q
a = 3 ** 0x10001
b = 5 ** 0x10001

_, x, y = extgcd(a, b)
m = pow(c1, x, N) * pow(c2, y, N) % N
flag = m.to_bytes((m.bit_length() + 7) // 8, "big")
```

第三份密文并非必要；甚至公开的 $p,q$ 也不是攻击所必需，只要知道 $N$ 即可。本地运行仓库中的 solver，得到：

```text
TSGCTF{You are intuitively understanding the distribution of prime numbers! Bonus: You can solve this challenge w/ N instead of p and q!}
```

## 方法总结

本题先用模 3 的简单结构确定秘密参数，再落到 RSA common modulus attack。只要同一明文在同一模数下使用两个互素指数加密，攻击者就能用 Bézout 系数组合两份密文直接恢复明文，不需要分解 $N$。设计 RSA 协议时不能把“指数很大”当作安全来源，更不能对同一裸明文重复使用可组合的指数；实际系统应使用标准随机填充方案。
