# Idk

## 题目简述

题目给出 RSA 公钥与密文，以及同一证明者生成的两份零知识证明 transcript。证明挑战 $\theta_j$ 由 $N$、索引和秘密字符串 $F$ 确定性哈希得到；两份 transcript 复用了相同 $F$，所以同一索引对应同一个 $\theta_j$。证明者计算模 $p$、模 $q$ 的平方根后，会随机翻转其中一个根的符号，导致两份证明可能泄露同一二次剩余的两个不同模 $N$ 平方根。

## 解题过程

若 $\mu_1^2\equiv\mu_2^2\equiv\theta\pmod N$ 且 $\mu_1\not\equiv\pm\mu_2\pmod N$，两个根在一个素因子下同号、另一个素因子下异号。因此：

$$
\gcd(\mu_1-\mu_2,N)
$$

会给出 $p$ 或 $q$。两份 dump 的格式都是一行 $F$、8 个 $\sigma_i$，随后 2840 个 $\mu_j$。按相同位置比较所有非零且不同的根：

```python
from Crypto.Util.number import GCD, inverse, long_to_bytes


def load_mus(path):
    lines = [line.strip() for line in open(path) if line.strip()]
    return [int(value, 16) for value in lines[1 + 8:1 + 8 + 2840]]


mus1 = load_mus("dump1.txt")
mus2 = load_mus("dump2.txt")

for left, right in zip(mus1, mus2):
    if left and right and left != right:
        factor = GCD(abs(left - right), N)
        if 1 < factor < N:
            p = factor
            q = N // factor
            break

phi = (p - 1) * (q - 1)
d = inverse(e, phi)
message = pow(c, d, N)
print(long_to_bytes(message))
```

本地运行官方数据可稳定分解 $N$，随后按普通 RSA 私钥恢复明文：

```text
grey{how_i_swear_you_shouldve_had_0_knowledge}
```

## 方法总结

- 核心技巧：从同一二次剩余的非平凡不同平方根取差并与 $N$ 求 gcd，直接分解 RSA 模数。
- 识别信号：证明挑战由可复用 transcript 标识确定，而平方根选择带随机符号时，多份证明会把 Rabin 型根歧义变成因子泄漏。
- 复用要点：只比较相同索引、相同 $F$ 的证明项，并排除零、相等以及 gcd 为 1 或 $N$ 的无效组合。
