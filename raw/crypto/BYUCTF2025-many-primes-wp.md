# Many Primes

## 题目简述

题目看似给出一个 4096 位 RSA 模数，但生成器并未使用两个大素数。它反复选取 10 到 16 位的小素数 $p$，随机取 2 到 5 的重数 $k$，并令 $n\leftarrow n\cdot p^k$，直到模数达到 4096 位；公钥指数为 $e=65537$。

模数很大并不代表难以分解，决定难度的是素因子的大小。

## 解题过程

所有素因子均小于 $2^{16}$，直接对奇数试除即可完整分解。若

$$
n=\prod_i p_i^{k_i},
$$

则欧拉函数为

$$
\varphi(n)=\prod_i p_i^{k_i-1}(p_i-1).
$$

```python
from Crypto.Util.number import long_to_bytes

left = n
phi = 1
for p in range(3, 2**16, 2):
    if left % p:
        continue
    k = 0
    while left % p == 0:
        left //= p
        k += 1
    phi *= p ** (k - 1) * (p - 1)

assert left == 1
d = pow(e, -1, phi)
print(long_to_bytes(pow(c, d, n)))
```

用 `assert left == 1` 确认没有漏掉因子，再计算私钥指数并解密，得到：

```text
byuctf{3ulers_ph1_function_15_v3ry_us3ful_4nd_th15_I5_a_l0ng_fl4g}
```

## 方法总结

- 核心技巧：利用生成器给出的平滑模数结构，以小范围试除恢复完整素因子分解。
- 识别信号：超大 RSA 模数由许多很小且重复的素数相乘时，位数只是表象，试除成本由最大素因子决定。
- 复用要点：含重复因子时不能只乘 $(p-1)$，必须使用 $p^{k-1}(p-1)$；分解后应验证剩余商为 1。
