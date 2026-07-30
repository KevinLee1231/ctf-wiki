# L3akCTF 2024 RSAn’t Writeup

## 题目简述

题目没有直接公开 RSA 模数 $N$，只给出 flag 密文以及两条已知明文的 RSA 密文。素数生成又共享同一个大整数 `tmp`：

```python
p = next_prime(1337 * tmp + random_512_bits)
q = next_prime(7331 * tmp + random_512_bits)
```

解法分两步：先利用已知明密文对恢复 $N$，再利用 $p,q$ 的近似线性关系和 512 位小误差，通过 Coppersmith 小根分解模数。

## 解题过程

### 恢复未公开的模数

对已知明文 $m_i$ 与密文 $c_i$，存在整数 $k_i$ 使：

$$
m_i^e-c_i=k_iN.
$$

因此：

$$
\gcd(m_1^e-c_1,\ m_2^e-c_2)
$$

是 $N$ 的整数倍。官方脚本得到 GCD 后继续试除 2 到 999 的小因子，清除共同商带来的小倍数，恢复真实模数。

```python
candidate = gcd(m1**e - c1, m2**e - c2)
for factor in range(2, 1000):
    while candidate % factor == 0:
        candidate //= factor
N = candidate
```

### 利用相关素数分解

忽略 512 位误差时：

$$
N\approx(1337t)(7331t),
$$

所以可估计：

$$
t\approx\sqrt{\frac{N}{1337\cdot7331}}.
$$

以此构造 $q$ 的近似值 $q_0$，把真实素数写成 $q=q_0+x$，其中 $x$ 的范围约为 512 位。然后在 $\mathbb Z_N[X]$ 上对：

$$
f(X)=X+q_0
$$

调用 `small_roots(X=2^512)`。找到根后即可得到 $q$，再计算 $p=N/q$、$\varphi(N)$ 和私钥指数：

```sage
t = isqrt(N // (1337 * 7331))
q0 = 7331 * t - 2**512

R.<x> = PolynomialRing(Zmod(N), implementation="NTL")
for delta in (x + q0).small_roots(X=2**512, beta=0.1):
    q = int(q0 + delta)
    if N % q == 0:
        p = N // q
        d = inverse_mod(e, (p - 1) * (q - 1))
        print(long_to_bytes(pow(ciphertext, d, N)))
```

仓库没有单独保存明文 flag，但 `dist/sol.sage` 已包含完整输出参数和上述官方求解过程，因此该题的利用链证据充分；不能根据题名或密文猜测 flag 内容。

## 方法总结

- 核心技巧：由已知 RSA 明密文对取 GCD 恢复隐藏模数，再用共享近似参数形成的部分素数信息执行 Coppersmith 小根攻击。
- 识别信号：模数未公开不代表无法恢复；只要存在两组以上已知明密文，就应检查 $m^e-c$ 的 GCD。
- 复用要点：GCD 结果可能是 $N$ 的小倍数；构造小根多项式前应结合素数生成公式估计误差界，并在得到候选根后用整除关系验证。
