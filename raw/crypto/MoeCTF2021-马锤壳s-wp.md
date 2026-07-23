# 马锤壳s

## 题目简述

题目把 flag 解释为整数，再将其二进制表示左侧补零到 $17\times17=289$ 位，按行填入矩阵并转置，得到只含 $0/1$ 的矩阵 $M$。随后生成一个由 256 位素数组成的 $17\times17$ 矩阵 $K$，在整数环上计算

$$
C=MK.
$$

输出除 $C$ 外，还泄露了矩阵

$$
L=65537K+12138\pmod {2^{512}}.
$$

核心问题是先从仿射泄露 $L$ 恢复 $K$，再解矩阵方程取回 $M$。

## 解题过程

### 1. 从仿射泄露恢复密钥矩阵

$65537$ 是奇数，因此

$$
\gcd(65537,2^{512})=1,
$$

它在模 $2^{512}$ 的剩余类环中存在乘法逆元。对每个矩阵元素都有

$$
K_{i,j}\equiv (L_{i,j}-12138)\cdot65537^{-1}\pmod {2^{512}}.
$$

源码中 $K_{i,j}$ 只有 256 位，乘以 65537 再加 12138 仍远小于 $2^{512}$，所以这里不会发生回绕；恢复出的剩余类就是原始素数本身。

### 2. 解出二进制矩阵

已知 $C=MK$，可在 SageMath 中调用 `K.solve_left(C)` 求满足 $XK=C$ 的左解。解出的矩阵应全部为 $0$ 或 $1$；转置后按行展开，正好撤销题目生成明文矩阵时的转置。

把题目输出中的两个矩阵分别整理为 Python 行列表后，可使用：

```python
from sage.all import Matrix, ZZ, inverse_mod
from Crypto.Util.number import long_to_bytes

# 用题目 output 中的 17×17 矩阵替换这两个占位符。
C_ROWS = [...]
LEAK_ROWS = [...]

MOD = 2**512
A = 0x10001
B = 12138
A_INV = inverse_mod(A, MOD)

K_ROWS = [
    [
        ZZ(((LEAK_ROWS[i][j] - B) * A_INV) % MOD)
        for j in range(17)
    ]
    for i in range(17)
]

K = Matrix(ZZ, K_ROWS)
C = Matrix(ZZ, C_ROWS)
M = K.solve_left(C)  # 返回满足 M*K=C 的左解

assert M.nrows() == M.ncols() == 17
assert all(bit in (0, 1) for bit in M.list())

bits = "".join(str(bit) for bit in M.transpose().list())
flag = long_to_bytes(int(bits, 2))
print(flag)
```

输出为：

```text
moectf{L1ne4r_Alg3br4_w1ll-b3,c0o1~}
```

## 方法总结

- 核心技巧：利用奇数在模 $2^n$ 环中可逆，逐元素撤销矩阵的仿射掩码，再求矩阵左解。
- 识别信号：看到 $AK+B\bmod 2^n$ 时，先检查 $\gcd(A,2^n)$；$A$ 为奇数即可直接求逆。
- 复用要点：从模环转回整数前要确认是否发生回绕；最后还要按源码顺序撤销转置、补零和位串展开。
