# Phish

## 题目简述

WelcomeCTF2021 的 Phish 给出 $A=p_1p_2p_3p_4$ 及其欧拉函数

$$
\varphi(A)=\prod_{i=1}^{4}(p_i-1),
$$

还给出另一组 RSA 参数 $N,c$。RSA 的两个素数由四个 $p_i$ 中的某一个作为 Python `random` 的种子生成。目标是先借助已知 $\varphi(A)$ 分解 $A$，再复现 RSA 密钥。

## 解题过程

对于与 $A$ 互素的底数 $g$，幂 $g^k\bmod A$ 在不同素因子模数下的阶不同。如果选取 $k$ 为 $\varphi(A)$ 除去一部分 2 因子后的值，某些 $p_i-1$ 仍整除 $k$，另一些则不整除。于是

$$
\gcd(g^k-1,A)
$$

有机会给出非平凡因子。

官方脚本根据当前剩余素因子个数依次取 $d=16,8,4,2$，尝试不同底数：

```python
primes = []

for i in range(4):
    divisor = 2 ** (4 - i)
    base = 2
    while True:
        factor = math.gcd(pow(base, phi // divisor, A) - 1, A)
        if is_prime(factor):
            break
        if is_prime(A // factor):
            factor = A // factor
            break
        base += 1

    primes.append(factor)
    A //= factor
    phi //= factor - 1
```

每找到一个素因子，就同时从 $A$ 中除去 $p_i$、从 $\varphi(A)$ 中除去 $p_i-1$，再处理剩余部分。

随后逐个尝试四个恢复出的种子。题目使用确定性的 `random.seed(seed)`，并以两次 `randint` 的结果取 `next_prime`：

```python
for seed in primes:
    random.seed(seed)
    p = next_prime(random.randint(1 << 2047, 1 << 2048))
    q = next_prime(random.randint(1 << 2047, 1 << 2048))
    if p * q != N:
        continue

    d = pow(0x10001, -1, (p - 1) * (q - 1))
    flag = long_to_bytes(pow(c, d, N))
```

匹配 $p q=N$ 的种子即为正确种子，解密得到：

```text
greyhats{F@ct0r1n9_w1th_ph1}
```

## 方法总结

已知合数及其欧拉函数通常足以显著削弱分解难度。本题把这种信息泄露与可复现的非密码学 PRNG 串在一起：先用幂与最大公约数拆出 $A$ 的素因子，再重放随机数生成过程。两层中的任一层使用安全参数，都能阻断整条攻击链。
