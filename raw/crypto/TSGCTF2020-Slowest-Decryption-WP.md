# TSGCTF2020 Slowest Decryption WP

## 题目简述

朴素解密函数对长度 $n=20000$ 的数组 $a$ 枚举所有 $n^n$ 个序列 $P=(p_0,\ldots,p_{n-1})$，计算：

$$
D=\sum_{P\in\{0,\ldots,n-1\}^n}
\gcd(p_0,\ldots,p_{n-1})
\sum_{i=0}^{n-1}i\,a_{p_i}\pmod M.
$$

直接运行显然不可行。关键是按 $\gcd(P)$ 分组，把“gcd 恰好等于 $k$”通过“所有元素都是 $k$ 的倍数”进行倍数反演，将指数枚举降为约 $O(n\log n)$。

## 解题过程

令 $f(k)$ 表示所有满足 $\gcd(P)=k$ 的序列对应的加权和。最终答案为：

$$
D=\sum_{k=1}^{n-1}k f(k).
$$

先放宽条件，只要求每个 $p_i$ 都能被 $k$ 整除。可选值集合为：

$$
\{0,k,2k,\ldots,\lfloor(n-1)/k\rfloor k\},
$$

大小 $m_k=\lfloor(n-1)/k\rfloor+1=\lceil n/k\rceil$。固定位置 $i$ 和某个值 $jk$ 后，其余 $n-1$ 个位置有 $m_k^{n-1}$ 种取法，而 $\sum_i i=n(n-1)/2$。因此所有元素均为 $k$ 倍数时的总和为：

$$
G(k)=m_k^{n-1}\frac{n(n-1)}2
\sum_{j=0}^{\lfloor(n-1)/k\rfloor}a_{jk}.
$$

$G(k)$ 同时包含 gcd 为 $k,2k,3k,\ldots$ 的序列，以及全零序列。全零序列的 gcd 按 Python 计算为 0，不能计入任何正 $f(k)$，但其内层和为 $C_0=\frac{n(n-1)}2a_0$。于是：

$$
f(k)=G(k)-C_0-
\sum_{m=2}^{\lfloor(n-1)/k\rfloor}f(mk).
$$

从 $k=n-1$ 递减到 1 计算，就能直接使用所有已知的更大倍数项。对应实现为：

```python
C = n * (n - 1) // 2
zero = C * a[0] % MOD
f = [0] * n

for k in range(n - 1, 0, -1):
    count = (n - 1) // k + 1
    divisible_sum = sum(a[0:n:k])
    G = pow(count, n - 1, MOD) * C * divisible_sum
    f[k] = (G - zero - sum(f[2*k:n:k])) % MOD

answer = sum(k * f[k] for k in range(1, n)) % MOD
```

总遍历量为 $\sum_{k=1}^{n-1}n/k=O(n\log n)$；[官方推导](https://scrapbox.io/tsgctf2-writeup-naan/Slowest_Decryption_writeup_(En))还给出了通过筛法合并倍数和的 $O(n\log\log n)$ 实现，即仓库 `solver.py` 中的 `coef/cnt` 更新。将结果整数按大端转回字节，仓库数据实际解出：

```text
TSGCTF{GRE4T!_y0u_Found_n1c3_decription_Alg0r1thm_or_you_h4ve_aston1shing_Fa5t_c4lcul4t0r}
```

根目录 `writeup.md` 把其中的 `decription` 写成了 `decryption`，但对 `encrypted.json` 运行官方 solver 得到的是上面的仓库实际字符串。

## 方法总结

本题把不可行的 $n^n$ 枚举变成整除偏序上的反演。先计算“gcd 是 $k$ 的倍数”的易求总和，再减去所有严格倍数的精确贡献，就得到“gcd 恰为 $k$”。固定一个位置和值、计数其余自由位置，是化简内层加权和的关键。看到 gcd、lcm 或整除条件上的全空间求和时，应优先尝试按约数/倍数分组和 Möbius 型反演，而不是优化原始笛卡尔积枚举。
