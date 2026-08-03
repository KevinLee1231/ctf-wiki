# too many primes

## 题目简述

这是一个多素数 RSA。生成器先随机选择 128 位素数 $p_0$，随后不断取 `nextprime`：

```python
p = randprime(2**127, 2**128)
N = 1
while N < 2**2048:
    N *= p
    p = nextprime(p)
```

最终模数由一串连续素数组成，再计算 $c=m^{65537}\bmod N$。多素数 RSA 本身并不必然不安全；这里的缺陷是所有因子大小近似且彼此极近，使 $N$ 的因子可以从整数高次根附近直接枚举。

## 解题过程

仓库实例实际包含 17 个因子。若：

$$
N=\prod_{i=0}^{16}p_i
$$

且所有 $p_i$ 都接近同一个 128 位数，那么：

$$
\sqrt[17]{N}\approx p_i.
$$

先计算 $t=\lfloor\sqrt[17]{N}\rfloor$，再在 $t$ 附近测试整除性即可。对本实例搜索 $\pm10000$ 的窗口就找齐 17 个因子；本地复核得到最大、最小因子之差仅为 2250。

```python
from math import prod
from sympy import integer_nthroot
from Crypto.Util.number import long_to_bytes

N = ...   # 填入 chal.py 注释中的模数
ct = ...  # 填入 chal.py 注释中的密文

center, exact = integer_nthroot(N, 17)
assert not exact

primes = [
    candidate
    for candidate in range(center - 10000, center + 10000)
    if N % candidate == 0
]

assert len(primes) == 17
assert prod(primes) == N

phi = prod(p - 1 for p in primes)
d = pow(65537, -1, phi)
flag = long_to_bytes(pow(ct, d, N))
print(flag)
```

恢复全部因子后，按多素数 RSA 计算：

$$
\varphi(N)=\prod_{i=0}^{16}(p_i-1),\qquad
d\equiv65537^{-1}\pmod{\varphi(N)}.
$$

解密结果为：

```text
uiuctf{D0nt_U5e_Cons3cUt1vE_PriMeS}
```

## 方法总结

- 核心技巧：对由大量近邻素数组成的模数取整数高次根，在根附近以整除测试恢复全部因子。
- 识别信号：多素数 RSA、因子由 `nextprime` 连续生成、所有因子位数相同，而且模数位数能够估计因子数量。
- 复用要点：先根据 $\operatorname{bitlen}(N)/\operatorname{bitlen}(p)$ 估计因子个数；搜索窗口是由“连续素数”结构支持的实例参数，不是任意多素数 RSA 的通用常数。最终必须验证因子乘积恰为 $N$。
