# SU_signin

## 题目简述

题目在 BLS12-381 的基域上使用曲线

$$
E/\mathbb F_p:y^2=x^3+4,
$$

并选择两个阶均为 $o$ 的点 $G_1,G_2$。再取

$$
P_1=\frac{o}{n_1}G_1,\qquad
P_2=\frac{o}{n_2}G_2,
$$

其中 $P_1$、$P_2$ 分别落入小阶子群。每个 flag bit 被编码为：

```python
# bit == 1
a * P1 + b * G2

# bit == 0
a * G1 + b * P2
```

所有点坐标公开。目标是利用子群结构与 Weil pairing 判断每个点来自哪一种分布。

## 解题过程

### 从已知首位构造判别点

flag 是 ASCII 字节串，最高位为 $0$；`zfill(bit_length)` 又保留了这一位。因此第一个密文点 $C_0$ 必定来自 bit 0 分支：

$$
C_0=aG_1+bP_2.
$$

由于 $P_2$ 的阶为 $n_2$，计算

$$
Q=n_2C_0=n_2aG_1
$$

即可消去小阶分量。

### 用配对是否为单位元区分两类点

对于另一个 bit 0 点 $C=xG_1+yP_2$，Weil pairing 的交替性给出 $e(G_1,G_1)=1$，双线性又给出：

$$
\begin{aligned}
e_o(C,Q)
&=e_o(xG_1+yP_2,n_2aG_1)\\
&=e_o(G_1,G_1)^{x n_2a}
  e_o(G_2,G_1)^{y(o/n_2)n_2a}\\
&=e_o(G_2,G_1)^{yoa}=1.
\end{aligned}
$$

若 $C=xP_1+yG_2$ 来自 bit 1 分支，$P_1$ 部分同样与 $Q$ 配对为 $1$，但 $G_2$ 部分通常留下

$$
e_o(G_2,G_1)^{y n_2a}\ne1.
$$

于是可逐点恢复比特：

```sage
K = GF(p)
E = EllipticCurve(K, (0, 4))
cs = [E(xy) for xy in cs]

Q = n2 * cs[0]
bits = "0"
for C in cs[1:]:
    value = C.weil_pairing(Q, o)
    bits += "0" if value == 1 else "1"

flag = long_to_bytes(int(bits, 2))
print(flag)
```

随机系数退化为零等极小概率事件可能让某一位误判；重新生成题目实例或结合已知 flag 格式复核即可。正常实例中，单位元测试可以直接还原完整 flag。

## 方法总结

- 核心技巧：用已知明文位提供的样本消去小阶分量，再通过 Weil pairing 检测另一个点是否含有独立的大阶方向。
- 识别信号：同一消息的两类编码分别混合“大阶点 + 不同小阶子群点”，而且曲线支持双线性配对时，应尝试乘子群阶并构造配对判别器。
- 复用要点：配对攻击依赖交替性、双线性和各点的精确阶；先写出每个分量在乘阶与配对后的去留，再实现单位元测试，避免仅凭“配对友好曲线”猜测。
