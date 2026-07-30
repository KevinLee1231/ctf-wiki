# L3akCTF 2024 Matrix Magic Writeup

## 题目简述

题目在未知合数模数 $n=pq$ 上构造一个 $6\times6$ 上三角分块矩阵，并输出 $C=M^m$，其中指数 $m$ 就是 flag 的大整数表示。矩阵对角线由三个 $2\times2$ Jordan 块组成：

$$
J_\lambda=
\begin{pmatrix}
\lambda&1\\
0&\lambda
\end{pmatrix},
\qquad \lambda\in\{2,4,8\}.
$$

虽然随机的右上角元素制造了大量噪声，但对角块的幂具有封闭公式，足以恢复模数和指数。

## 解题过程

Jordan 块的 $m$ 次幂为：

$$
J_\lambda^m=
\begin{pmatrix}
\lambda^m&m\lambda^{m-1}\\
0&\lambda^m
\end{pmatrix}.
$$

因此从输出矩阵可直接读到：

$$
x=C_{0,0}=2^m\bmod n,
\quad
y=C_{2,2}=4^m\bmod n,
\quad
z=C_{4,4}=8^m\bmod n.
$$

它们满足 $x^2\equiv y\pmod n$、$x^3\equiv z\pmod n$ 和 $y^3\equiv z^2\pmod n$。于是下列整数都含有模数 $n$ 作为因子：

```python
candidate = gcd(
    gcd(x**2 - y, x**3 - z),
    y**3 - z**2,
)
```

这个最大公因数可能是 $n$ 的小整数倍。官方 solver 从 3000 向下试除小因子，直到留下约 1024 位的真实模数。

再看第一个 Jordan 块的右上角：

$$
C_{0,1}=m2^{m-1}\pmod n.
$$

结合 $C_{0,0}=2^m$，可直接得到：

$$
m\equiv 2C_{0,1}C_{0,0}^{-1}\pmod n.
$$

flag 的整数长度远小于 $n$，所以该剩余类就是原始 $m$。转换为字节后得到：

```text
L3AK{jordan_normal_matrix_is_so_special_86351adeff}
```

## 方法总结

- 核心技巧：利用 Jordan 块幂的对角项与超对角项关系，从矩阵幂输出恢复未知模数和指数。
- 识别信号：未知模数上的矩阵幂若含多个相关特征值或 Jordan 块，应优先检查对角元素之间是否产生模数倍数。
- 复用要点：由整数关系取 GCD 时常得到模数的小倍数，需要结合预期位长和小因子试除清理；随机上三角噪声不影响对角块的不变量。
