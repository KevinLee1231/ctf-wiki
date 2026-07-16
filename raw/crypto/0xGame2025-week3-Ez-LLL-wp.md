# Ez_LLL

## 题目简述

题目把 flag 转为整数 $f$，生成 1024 位素数 $p$ 和约 350 位素数 $g$，然后公开：

$$
h\equiv g f^{-1}\pmod p.
$$

由于 $f$ 和 $g$ 都远小于 $p$，该模关系中隐藏着一个异常短的二维向量，可用 LLL 恢复。

## 解题过程

由公开关系可得：

$$
fh\equiv g\pmod p,
$$

即存在整数 $k$ 使 $fh-kp=g$。考虑由两行

$$
(1,h),\quad(0,p)
$$

生成的格，线性组合 $f(1,h)-k(0,p)$ 恰好是 $(f,g)$。它的两个坐标只有数百位，而格中一般向量受 1024 位模数控制，因此 $(f,g)$ 是显著短向量。

```sage
from sage.all import Matrix, ZZ
from Crypto.Util.number import long_to_bytes

p = ...
h = ...

basis = Matrix(ZZ, [
    [1, h],
    [0, p],
])

reduced = basis.LLL()
for vector in reduced.rows():
    candidate = long_to_bytes(abs(int(vector[0])))
    if candidate.startswith(b'0xGame{'):
        print(candidate)
```

LLL 可能返回短向量的相反数，所以第一坐标要取绝对值。用仓库参数运行得到：

```text
0xGame{8dc1f4b8-3f4e-4c3e-9d1a-2b5e6f7a8b9c}
```

## 方法总结

- 模关系 $fh\equiv g\pmod p$ 可改写成包含 $(f,g)$ 的整数格。
- 当未知量远小于模数时，应检查它们是否组成格中的异常短向量。
- LLL 结果只确定到符号和近似最短基，解码前应检查正负两种方向并验证 flag 格式。
