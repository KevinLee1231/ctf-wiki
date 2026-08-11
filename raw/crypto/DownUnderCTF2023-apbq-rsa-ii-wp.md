# DownUnderCTF 2023 apbq rsa ii Writeup

## 题目简述

第二题仍公开 $n=pq$、RSA 密文和线性提示 $h_i=a_ip+b_iq$，但现在有三个提示，且 $a_i,b_i$ 都扩展到 312 位，无法直接穷举。三个提示组成的向量位于两个短系数向量张成的平面中，可以用正交格恢复该平面的短基。

## 解题过程

记：

$$
\mathbf h=p\mathbf a+q\mathbf b,
$$

其中 $\mathbf a,\mathbf b\in\mathbb Z^3$ 的坐标约为 312 位。向量 $\mathbf r=\mathbf a\times\mathbf b$ 同时与两者正交，所以满足精确关系 $\mathbf r\cdot\mathbf h=0$。第一轮 LLL 在放大后的提示向量旁附加单位阵，找出这个整数关系；第二轮 LLL 再求与 $\mathbf r$ 正交的短向量，得到 $\mathbf a,\mathbf b$ 所在平面的短基。

下面保留了官方 Sage 解法的核心构造：

```python
from itertools import product
from Crypto.Util.number import long_to_bytes

exec(open("output.txt", "r").read())

values = hints
scale = 2**800

# 找到 h 的短整数正交关系。
lattice = Matrix.column([scale * value for value in values])
lattice = lattice.augment(identity_matrix(len(values)))
reduced = [row[1:] for row in lattice.LLL()]

# 在该关系的正交平面中恢复短基。
lattice = (scale * Matrix(reduced[:len(values) - 2])).T
lattice = lattice.augment(identity_matrix(len(values)))
basis = [
    row[-len(values):]
    for row in lattice.LLL()
    if set(row[:len(values) - 2]) == {0}
]

factor = None
for s, t in product(range(4), repeat=2):
    coeffs = s * basis[0] + t * basis[1]
    a0, a1, _ = coeffs
    candidate = gcd(a0 * hints[1] - a1 * hints[0], n)
    if 1 < candidate < n:
        factor = int(candidate)
        break

assert factor is not None
q = factor
p = n // q
d = pow(0x10001, -1, (p - 1) * (q - 1))
print(long_to_bytes(pow(c, d, n)).decode())
```

恢复出的 flag 为：

```text
DUCTF{0rtho_l4tt1c3_1s_a_fun_and_gr34t_t3chn1que_f0r_the_t00lbox!}
```

## 方法总结

当线性提示的系数不再可穷举时，可以利用“提示向量由少量短向量线性生成”的几何结构。先求精确正交关系，再在正交补中找短基，最后枚举极少量基组合并沿用第一题的消元与 `gcd`，即可完成分解。
