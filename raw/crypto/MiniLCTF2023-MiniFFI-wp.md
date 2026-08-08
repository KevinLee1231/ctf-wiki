# MiniLCTF2023 - MiniFFI

## 题目简述

题目在 $GF(41)[x]$ 中给出两个 128 次不可约多项式 $f,F$，从而得到两个同阶有限域

$$
K_f=GF(41)[x]/(f),\qquad K_F=GF(41)[x]/(F).
$$

多项式 $\phi$ 满足 $f(\phi(x))\equiv0\pmod F$，因此代入 $x\mapsto\phi(x)$ 给出 $K_f\to K_F$ 的同构。密文为

$$
C=2\phi\,r(\phi)+m(\phi)\pmod F,
$$

其中 $m,r$ 的系数只有 0 或 1。需要构造该有限域同构的逆映射，把密文拉回 $K_f$，再利用系数奇偶性提取 $m$。

## 解题过程

设逆映射中 $x$ 的像为

$$
\psi(x)=\sum_{i=0}^{127}c_i x^i.
$$

要求 $\psi(\phi(x))\equiv x\pmod F$。预先计算 $1,\phi,\ldots,\phi^{127}$ 在基 $1,x,\ldots,x^{127}$ 下的系数，组成 $128\times128$ 矩阵 $B$；随后解线性方程，使组合结果只在 $x^1$ 的位置为 1。这样就直接得到 $\psi$ 的系数。

下面保留完整算法结构；`f`、`F`、`phi`、`C` 与 `cipher` 是附件给出的长常量，原样粘入相应位置即可。

```python
from Crypto.Cipher import AES
from Crypto.Util.number import long_to_bytes
from sage.all import GF, Matrix, PolynomialRing, Zmod

p = 41
n = 128
R = PolynomialRing(GF(p), "x")
x = R.gen()

# f = ...
# F = ...
# phi = ...
# C = ...
# cipher = ...
assert f(phi) % F == 0

powers = [pow(phi, i, F) for i in range(n)]
B = Matrix(GF(p), n, n)
for i, poly in enumerate(powers):
    coeffs = poly.list() + [0] * (n - len(poly.list()))
    for j in range(n):
        B[i, j] = coeffs[j]

target = Matrix(GF(p), 1, n)
target[0, 1] = 1
psi = R(list(target * B.inverse())[0])
assert psi(phi) % F == x

pulled_back = (C(psi) % f).change_ring(Zmod(2))
bits = pulled_back.list()
bits += [0] * (n - len(bits))
key = int("".join(map(str, bits)), 2)

aes = AES.new(long_to_bytes(key, 16), AES.MODE_ECB)
print(aes.decrypt(cipher))
```

同一个逆映射也可以通过在 $K_f$ 中求 $F$ 的根，或求 $\phi(y)-x$ 的根得到，但必须选中与给定 $\phi$ 配对的根。官方材料未记录最后一次 AES 输出，因此这里保留可复现算法，不编造 flag 回显。

## 方法总结

有限域同构题最稳定的通用方法是把“代入多项式”视为基变换：每个 $\phi^i$ 都是一列或一行坐标，逆映射就是一次矩阵求逆。恢复后，$2xr+m$ 的系数模 2 只剩 $m$。实现时最容易出错的是矩阵行列方向、常数项与 $x$ 项的位置，以及 Sage 多项式稀疏系数未补足到固定长度。
