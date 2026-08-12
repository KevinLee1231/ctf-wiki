# DownUnderCTF 2021 - 1337crypt v2

## 题目简述

题目生成两个 1337 位素数 $p,q$，但不直接给出模数 $n=pq$。第一组提示是 $D=p^2+q^2$；第二组提示包含两次带有 338 位隐藏小数部分的复数范数取整结果。flag 被放入高斯整数环 $(\mathbb Z/n\mathbb Z)[i]$ 的虚部，再以 $e=\mathtt{0x1337}$ 次幂加密。

对第 $j$ 组提示，公开量为 $(y_j,a_j,b_j)$，而内部实际构造近似为：

$$
x_j=(a_j+2^{-l}c_j)+(b_j+2^{-l}d_j)(p+qi),\qquad
y_j=\lfloor x_j\overline{x_j}\rfloor,
$$

其中 $l=338$，$c_j,d_j<2^l$。缺失的小数位很短，可以把两组近似关系组合成格中的短向量。

## 解题过程

展开 $x_j\overline{x_j}$ 并代入 $D=p^2+q^2$。除以约有 2674 位的 $2a_jb_j$ 后，低阶项只造成约 338 位的误差，主要关系可写为：

$$
p\approx\frac{y_j-(b_j^2+2\cdot2^{-l}b_jd_j)D}{2a_jb_j}.
$$

两组关系中的 $p$ 相同。令

$$
t_j=\left\lfloor\frac{b_j^2D-y_j}{2a_jb_j}\right\rfloor,
\qquad
s_j=\left\lfloor\frac{2^{-l}D}{a_j}\right\rfloor,
$$

则隐藏的 $d_1,d_2$ 与一个小误差 $k$ 满足线性近似。构造行格：

$$
M=
\begin{pmatrix}
s_1&1&0\\
s_2&0&1\\
t_1-t_2&0&0
\end{pmatrix}.
$$

向量 $(k,-d_1,d_2)$ 是该格中的短向量，LLL 可以恢复其中一个 $d_j$。随后用上式估计 $p$；近似只差少量低位，在估计值附近枚举并检查 $D-p^2$ 是否为完全平方即可同时恢复 $p,q$。

核心 Sage 求解代码为：

```sage
from Crypto.Util.number import long_to_bytes

exec(open('../challenge/output.txt').read())

l = 338
delta = 2^-l
D = hint1

t1, t2 = [int(b^2 * D - y) // int(2*a*b)
          for y, a, b in hint2]
s1, s2 = [int(delta * D) // a for _, a, b in hint2]

M = Matrix([
    [s1, 1, 0],
    [s2, 0, 1],
    [t1 - t2, 0, 0],
])
d_hidden = abs(M.LLL()[0][1])

y, a, b = hint2[0]
p0 = int(y - (b^2 + 2*delta*b*d_hidden)*D) // int(2*a*b)

for p in range(p0 - 2^8, p0 + 2^8):
    if ZZ(D - p^2).is_square():
        q = ZZ(D - p^2).sqrt()
        break

n = p*q
Zn.<I> = (ZZ.quo(n*ZZ))[]
ZnI.<I> = Zn.quo(I^2 + 1)
c = ZnI(c)

d = inverse_mod(0x1337, (p-1)*(q-1))
m = c^d
print(long_to_bytes(list(m)[1]).decode())
```

恢复虚部后得到：

```text
DUCTF{mantissa_in_crypto??_n0_th4nks!!}
```

## 方法总结

本题的关键不是直接分解 $p^2+q^2$，而是利用两份“高精度数值被截断”的相关样本。展开范数、按数量级舍去低阶项、消去共同的 $p$ 后，隐藏的 338 位尾数形成低维格中的短向量；LLL 恢复尾数，再通过完全平方判定校正素数的少量低位。以后看到多个共享秘密、只缺少较短尾数的高精度近似时，应优先尝试把误差写成小根或短向量问题。
