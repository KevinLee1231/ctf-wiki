# Polynomial

## 题目简述

服务把 flag 的每个字节依次当作多项式系数，并在有限域模数 $p=2^{16}+1=65537$ 下计算：

$$
f(x)=\sum_{i=0}^{n-1}m_i x^i \pmod p.
$$

客户端可以选择任意整数 $x$ 并取得 $f(x)$。同一个多项式在服务生命周期内保持不变，因此这些查询就是可自由选择横坐标的插值样本。

## 解题过程

次数小于 $n$ 的多项式由 $n$ 个横坐标互异的点唯一确定。取 $x_j=j+1$，可得到线性方程组

$$
\begin{bmatrix}
1&x_0&x_0^2&\cdots&x_0^{n-1}\\
1&x_1&x_1^2&\cdots&x_1^{n-1}\\
\vdots&\vdots&\vdots&&\vdots\\
1&x_{n-1}&x_{n-1}^2&\cdots&x_{n-1}^{n-1}
\end{bmatrix}
\begin{bmatrix}
m_0\\m_1\\\vdots\\m_{n-1}
\end{bmatrix}
=
\begin{bmatrix}
f(x_0)\\f(x_1)\\\vdots\\f(x_{n-1})
\end{bmatrix}
\pmod{65537}.
$$

这是 Vandermonde 矩阵；所选横坐标在域中互异，所以矩阵可逆。题面没有直接给出 flag 长度，可以取一个明显大于常见 flag 长度的上界，例如 64。真实多项式的高次系数会解成零，再按第一个零字节截断：

~~~python
from pwn import remote
from sage.all import GF, Matrix, vector

COUNT = 64
F = GF(65537)
io = remote(HOST, PORT)

xs = list(range(1, COUNT + 1))
ys = []
for x in xs:
    io.sendlineafter(b"> ", str(x).encode())
    ys.append(int(io.recvline().split()[-1]))

vandermonde = Matrix(
    F,
    [[F(x) ** i for i in range(COUNT)] for x in xs],
)
coeffs = vandermonde.solve_right(vector(F, ys))
plain = bytes(int(value) for value in coeffs)
print(plain.split(b"\x00", 1)[0].decode())
~~~

恢复出的系数顺序就是原始字节顺序，不需要反转：

~~~text
shellmates{___Lagrange__w4s__4__G3n1us___}
~~~

## 方法总结

任意点求值 oracle 会把“自制加密”直接变成有限域多项式插值问题。识别信号是固定系数、攻击者可选 $x$、返回 $\sum m_i x^i\bmod p$。只要采集足够多的互异点，使用 Lagrange 插值或解 Vandermonde 线性方程组都能恢复全部系数；未知长度可以通过多取样并检查尾部零系数解决。
