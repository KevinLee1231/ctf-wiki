# matrix

## 题目简述

题目先用 SHAKE256 把输入映射成 $4\times4$ 矩阵，矩阵元素位于 $\mathbb Z/256\mathbb Z$，然后执行五轮：

$$
X_{i+1}=X_iA+B.
$$

附件公开了矩阵 $A$，并给出一条已知消息经过该变换后的结果 $C$。固定但未知的矩阵 $B$ 是恢复另一条消息哈希值的关键。

## 解题过程

把 $B$ 的 16 个元素设为未知量 $b_0,\ldots,b_{15}$。已知消息经 SHAKE256 和 `number_to_matrix` 得到初始矩阵 $X_0$，按源码展开五轮后，每个位置都应与输出矩阵 $C$ 的对应元素相等：

$$
(X_5)_{i,j}-C_{i,j}=0\pmod{256}.
$$

官方 Sage 解法在 $\mathbb Z_{256}$ 上建立 16 元多项式环并求 Gröbner 基：

```sage
R = PolynomialRing(Zmod(256), "b", 16)
variables = R.gens()
B = Matrix(R, 4, 4, variables)

X = number_to_matrix(shake256_as_integer(known_message))
for _ in range(5):
    X = X * A + B

equations = [X[i, j] - C[i, j] for i in range(4) for j in range(4)]
G = Ideal(equations).groebner_basis()
```

从基中读出 16 个变量的常数解，重建 $B$，再对目标表单 URL 执行相同的 SHAKE256、矩阵转换和五轮迭代，得到：

```text
greyhats{19a8b656314c2eefa4d43bdd2155984b}
```

## 方法总结

自定义哈希若把秘密参数线性地重复注入状态，单个已知输入输出对就可能形成足够多的方程。本题的重点不是攻击 SHAKE256，而是把其输出当作已知初始状态，针对后续矩阵迭代恢复 $B$。实现时必须与源码保持相同的元素顺序、模数和轮数。
