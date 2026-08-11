# DownUnderCTF 2023 Advanced Vector Expositions Writeup

## 题目简述

程序表面上是一个 20×20 的字符地图游戏：玩家可以移动，并在 16 个书签位置反复执行标记操作。真正的校验逻辑藏在 `userfaultfd` 缺页处理流程中。每标记一次第 $i$ 个书签，程序就在有限域 $\mathrm{GF}(251)$ 上把一个基矩阵对状态矩阵的贡献累加进去；只有最终状态等于目标矩阵，程序才会输出 flag。

因此本题的决定性障碍不是游戏操作，而是把 16 个书签的标记次数还原为一个矩阵方程的解。

## 解题过程

设第 $i$ 个书签被标记 $c_i$ 次，对应的 $4\times4$ 基矩阵为 $B_i$。源码中的累加操作等价于

$$
\text{curr}\leftarrow\text{curr}+A B_i+B_i B\pmod{251}.
$$

令

$$
X=\sum_{i=0}^{15}c_iB_i,
$$

那么通过校验所需的条件就是 Sylvester 方程

$$
AX+XB=C.
$$

按列展开矩阵，并使用恒等式 $\operatorname{vec}(AXB)=(B^T\otimes A)\operatorname{vec}(X)$，可以把方程改写成一个 16 维线性系统：

$$
\left(I\otimes A+B^T\otimes I\right)\operatorname{vec}(X)=\operatorname{vec}(C).
$$

官方 Sage 求解脚本的核心逻辑如下。所有矩阵都必须建立在 $\mathrm{GF}(251)$ 上，否则普通整数或浮点求解会得到错误结果。

```python
F = GF(251)
A = Matrix(F, A)
B = Matrix(F, B)
C = Matrix(F, C)
I = Matrix.identity(F, 4)
bases = [Matrix(F, base) for base in bases]

def vectorize(M):
    return vector(sum(map(list, M.columns()), []))

def unvectorize(v):
    return Matrix([v[i:i + 4] for i in range(0, 16, 4)]).T

L = I.tensor_product(A) + B.T.tensor_product(I)
X = unvectorize(L.solve_right(vectorize(C)))

base_vectors = [vectorize(base) for base in bases]
counts = Matrix(base_vectors).T.solve_right(vectorize(X))
print(counts)
```

第一步解出 $X$，第二步再把 $X$ 表示成 16 个基矩阵的线性组合。得到的书签标记次数依次为：

```text
[4, 3, 3, 8, 1, 2, 7, 6, 5, 8, 7, 3, 5, 9, 2, 0]
```

它们与地图坐标的对应关系为：

```text
(13,0)=4   (5,1)=3    (2,2)=3    (14,2)=8
(17,2)=1   (8,4)=2    (17,5)=7   (4,6)=6
(6,8)=5    (10,9)=8   (15,9)=7   (1,11)=3
(10,12)=5  (12,15)=9  (3,17)=2   (15,18)=0
```

在游戏中把各书签调整到这些次数后，站到一个上方不是书签的位置并按 `m` 触发最终检查。程序确认 `curr == C` 后，会从计数构造矩阵并计算输出，得到：

```text
DUCTF{m4ppy_m4tr1x_avx}
```

## 方法总结

本题用字符游戏和缺页处理隐藏了一个有限域线性代数问题。关键是先把每次操作的累加式整体化为 $AX+XB=C$，再通过 Kronecker 积将矩阵未知量向量化。解出目标矩阵后，还要完成第二层基变换，才能得到实际需要输入到地图中的 16 个计数。
