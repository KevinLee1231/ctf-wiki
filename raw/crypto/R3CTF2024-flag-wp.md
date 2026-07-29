# flag

## 题目简述

题目把 RSA 的一个素因子藏在 19 层循环的计数结果里。循环变量之和固定为 $2^{20}$，其中 `j` 固定为旗帜字符的 Unicode 码点，其余 18 个变量按指数 $1$ 到 $9$ 分成两组，取值范围均为 $[2^i,2^{i+10}]$。程序令满足全部条件的方案数为 $p$，再使用 `nextprime(p)` 作为 RSA 的一个素因子：

```python
n = nextprime(p) * getPrime(256)
c = pow(bytes_to_long(flag), 65537, n)
```

显然不能执行原始的 18 重有效枚举。关键是把计数问题转换为带上界的整数拆分，再用生成函数或容斥直接计算。

## 解题过程

先代入固定值：

```python
ord("🚩") == 128681
```

对每个自由变量减去下界。指数 $i$ 对应两个变量，每个新变量的范围变为 $[0,1023\cdot 2^i]$。减去全部下界后，新的目标和为

$$
T=2^{20}-128681-2\sum_{i=1}^{9}2^i=917851.
$$

因此，$p$ 是下面生成函数中 $x^T$ 的系数：

$$
\prod_{i=1}^{9}\left(1+x+\cdots+x^{1023\cdot2^i}\right)^2.
$$

把每个上界因子写成 $(1-x^{U_i+1})^2/(1-x)^2$，其中 $U_i=1023\cdot2^i$。对分子做容斥时，同一层的两个变量可以截断 0、1 或 2 次。令 $c_i\in\{0,1,2\}$，则方案数为

$$
p=\sum_{c_1,\ldots,c_9}
(-1)^{\sum c_i}
\left(\prod_{i=1}^{9}\binom{2}{c_i}\right)
\binom{T-\sum c_i(U_i+1)+17}{17},
$$

其中组合数上标小于 17 时该项按 0 处理。这里只有 $3^9=19683$ 项，可以直接精确求和：

```python
from itertools import product
from math import comb

target = 917851
p = 0

for choices in product(range(3), repeat=9):
    shifted = target
    coefficient = 1
    parity = 0

    for level, count in enumerate(choices, start=1):
        shifted -= count * (1023 * (1 << level) + 1)
        coefficient *= comb(2, count)
        parity += count

    if shifted >= 0:
        term = coefficient * comb(shifted + 17, 17)
        p += -term if parity & 1 else term

print(p)
```

计算得到：

```text
p = 2565991931871938384822588623964880023351125901093463977291141641724850219374
nextprime(p) = 2565991931871938384822588623964880023351125901093463977291141641724850219449
```

这个素数整除题目给出的 `n`。分解模数、计算私钥指数并解密：

```python
from Crypto.Util.number import long_to_bytes
from sympy import nextprime

P = int(nextprime(p))
Q = n // P
phi = (P - 1) * (Q - 1)
d = pow(65537, -1, phi)
print(long_to_bytes(pow(c, d, n)))
```

本地使用题目附件中的 `n`、`c` 验证，得到：

```text
R3CTF{Oh_Y0u_Found_th3_L4st_fl@g_4nd_H@v3_FUN!}
```

## 方法总结

这道题的表象是不可执行的深层循环，实质是“变量有上下界且总和固定”的组合计数。先平移下界，再把两个同界变量合并进容斥系数，便可把约 $2^{360}$ 量级的枚举压缩为 $3^9$ 项精确计算。得到计数后，`nextprime` 并未提供任何密码学隐藏性，直接恢复 RSA 素因子即可。上述计数值、素因子与明文均已用附件中的固定参数实际验证。
