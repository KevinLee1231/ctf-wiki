# DownUnderCTF 2021 - flag printer

## 题目简述

Go 程序看起来会直接打印 flag，但每输出一个字符前都要反复进行 $50\times50$ 矩阵乘法。循环次数由大整数 `y` 控制，且每轮执行 `y = y * y`；`y` 很快增长到无法实际迭代的规模。矩阵始终只是固定矩阵 $M$ 的幂，因此可在有限域上对 $M$ 对角化，把线性次数的重复乘法改为快速幂。

## 解题过程

### 化简原程序

矩阵元素均在素域 $GF(p)$ 中运算，其中：

```text
p = 3766999387
```

程序初始令 $X=M$。对数组 `xs` 中的每个 $x_i$，先执行 `y` 次 `X = X * M`，再执行 $x_i$ 次相同操作，随后输出全部矩阵元素之和模 127：

```go
for _, x := range xs {
    for countY.Cmp(y) < 0 {
        X = Mul(X, M)
        countY.Add(countY, big.NewInt(1))
    }
    for countX < x {
        X = Mul(X, M)
        countX++
    }
    y.Mul(y, y)
    fmt.Printf("%c", rune(Sum(X)))
}
```

因此一轮只是在当前指数上加 $y+x_i$：

$$
X\leftarrow X M^{y+x_i}.
$$

真正的障碍不是矩阵尺寸，而是直接循环计算 $M^{y+x_i}$。

### 对角化并快速计算矩阵幂

把源码中的 $M$、`xs` 和素数 $p$ 提取到 SageMath。该实例的矩阵可在 $GF(p)$ 上对角化：

$$
M=PDP^{-1},
$$

其中 $D$ 为对角矩阵。于是：

$$
M^k=PD^kP^{-1},
$$

而 $D^k$ 只需对 50 个对角元素分别做标量快速幂：

```sage
p = 3766999387
F = GF(p)
MS = MatrixSpace(F, 50)
M = MS(M_data)
D, P = M.diagonalization()

def fast_exp(k):
    diagonal = MS.diagonal_matrix([value^k for value in D.diagonal()])
    return P * diagonal * ~P
```

### 约减爆炸增长的 `y`

本题矩阵的特征值都非零。对任意特征值 $\lambda\in GF(p)^*$，费马小定理给出：

$$
\lambda^{p-1}=1.
$$

所以矩阵幂的指数只需保留模 $p-1$ 的值。原程序中的大整数 `y` 虽不断平方，但用于计算 $M^y$ 时可以安全更新为：

```python
y = y * y % (p - 1)
```

这一步成立依赖当前矩阵可对角化且没有零特征值；不能不检查条件就把同一结论套到任意矩阵。

### 重放字符输出

用矩阵 `current` 保存程序中的 $X$，保持原始初值 $X=M$，逐轮累乘快速计算出的矩阵幂：

```sage
def sum_entries(A):
    return sum(map(int, A.list())) % 127

y = 2
current = M
flag = ""

for x in xs:
    current *= fast_exp(y + x)
    flag += chr(sum_entries(current))
    y = y * y % (p - 1)

print(flag)
```

在显式激活的 SageMath 环境中运行仓库 solver，输出已核对为：

```text
DUCTF{g0tta_GO_f4sTtTTtt!11!1!!_1c5eff59b5e8814d7f92}
```

## 方法总结

本题将可化简的矩阵幂伪装成天文数量的重复矩阵乘法。识别出 $X$ 始终是同一个 $M$ 的幂后，可通过 $GF(p)$ 上的对角化把矩阵快速幂降为特征值快速幂，并利用非零特征值的乘法群阶 $p-1$ 约减指数。关键前提是实际检查矩阵的可对角化性和非零谱，而不是仅凭“有限域”三个字盲目套用费马小定理。
