# DUCTF 2023 apbq-rsa-ii

## 题目简述

题目是一个 RSA 变种。生成脚本先取 1024 bit 素数 `p, q`，令 `n = p*q`、`e = 0x10001`，再输出三条提示：

```python
hints.append(a * p + b * q)
```

其中每组 `a, b` 都是不超过 `2^312` 的随机整数。附件同时给出 `n`、密文 `c` 和三个 `hint`。目标是利用这些小系数线性提示分解 RSA 模数，再正常 RSA 解密。

## 解题过程

### 关键观察

三条提示形如：

```text
h1 = a1*p + b1*q
h2 = a2*p + b2*q
h3 = a3*p + b3*q
```

虽然 `a_i, b_i` 未知，但它们很小。可以构造一个格，让 LLL 寻找小整数关系，使提示之间的线性组合在模 `n` 意义下消掉前几项。现有解法使用 6 维格，并对前三列乘上较大的缩放因子，使 LLL 更容易返回形如 `(0, 0, 0, x1, x2, x3)` 的短向量。随后对这些 `x_i` 与 `n` 取 gcd，若得到非平凡因子，就完成分解。

### 求解步骤

Sage 脚本核心如下：

```python
from sage.all import *
from Crypto.Util.number import long_to_bytes

n = Integer(...)
c = Integer(...)
h1, h2, h3 = map(Integer, hints)
e = 0x10001

M = Matrix([
    [h2,  h3,  0,  1, 0, 0],
    [-h1, 0,   h3, 0, 1, 0],
    [0,  -h1, -h2, 0, 0, 1],
    [n,   0,   0,  0, 0, 0],
    [0,   n,   0,  0, 0, 0],
    [0,   0,   n,  0, 0, 0],
])

k = 2 ** (1024 + 312)
W = diagonal_matrix([k, k, k, 1, 1, 1])
R = (M * W).LLL() / W

for row in R:
    if row[0] == row[1] == row[2] == 0:
        for x in row[3:]:
            p = gcd(Integer(x), n)
            if 1 < p < n:
                q = n // p
                d = inverse_mod(e, (p - 1) * (q - 1))
                print(long_to_bytes(int(pow(c, d, n))))
                raise SystemExit
```

运行 `solve2.sage` 后恢复出 RSA 私钥并得到：

```text
DUCTF{0rtho_l4tt1c3_1s_a_fun_and_gr34t_t3chn1que_f0r_the_t00lbox!}
```

## 方法总结

- 核心技巧：把小系数 RSA 线性提示转化为短向量问题，通过 LLL 找到可用于分解 `n` 的关系。
- 识别信号：看到 `a*p + b*q` 这类提示，且 `a, b` 明显小于 `p, q` 时，要考虑格约简而不是直接求解线性方程。
- 复用要点：缩放因子决定 LLL 搜索方向；脚本应在找到候选关系后立即用 `gcd(x, n)` 验证，不能只凭短向量形态判断成功。
