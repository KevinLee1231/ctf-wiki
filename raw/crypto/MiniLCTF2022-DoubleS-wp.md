# MiniLCTF2022 Double S Writeup

## 题目简述

题目把去掉 `miniLCTF{}` 外壳后的 flag 放在两个 `#` 之间，并补齐到 64 字节，再按每 4 字节一组转成 16 个整数。随后追加 16 个随机系数，得到次数不超过 31 的整数多项式

$$
f(x)=a_0+a_1x+\cdots+a_{31}x^{31}.
$$

附件给出了 32 组互不相同的成员名与 $f(x)$ 值。决定性问题不是传统门限秘密共享，而是已知 32 个点后恢复全部 32 个系数。

## 解题过程

将成员名按题目相同的 `bytes_to_long` 规则转为 $x_i$，对应密文记为 $c_i$。对每一组数据都有

$$
\sum_{j=0}^{31}a_jx_i^j=c_i.
$$

于是构造 $32\times32$ 的 Vandermonde 矩阵 $V$：

$$
V_{i,j}=x_i^j,\qquad V\boldsymbol a=\boldsymbol c.
$$

运算发生在整数环而不是模环。用 SageMath 在有理数域上求解线性方程组，所得系数均为整数；前 16 个系数依次转回 4 字节并拼接，即可还原 64 字节秘密区。核心代码如下：

```python
from Crypto.Util.number import bytes_to_long, long_to_bytes

xs = [bytes_to_long(name) for name, _ in samples]
cs = [cipher for _, cipher in samples]

V = Matrix(QQ, [[x**j for j in range(32)] for x in xs])
a = V.solve_right(vector(QQ, cs))
assert all(v.denominator() == 1 for v in a)

secret = b"".join(long_to_bytes(int(v), 4) for v in a[:16])
body = secret.split(b"#", 2)[1]
print(b"miniLCTF{" + body + b"}")
```

恢复结果为：

```text
miniLCTF{y0u_c4n_s0lve_i7_bY_L1ne@r_Alg3br4_e4si1y~!}
```

## 方法总结

当次数至多为 $n-1$ 的多项式给出 $n$ 个不同横坐标上的取值时，系数被唯一确定。此题最容易误判的地方是把它当成模有限域上的 Shamir Secret Sharing；源码没有取模，因此应在 $\mathbb Q$ 上精确求解并检查结果是否为整数。恢复后还要严格沿用题目的 4 字节大端分组和 `#...#` 边界，避免把随机补位当作 flag。
