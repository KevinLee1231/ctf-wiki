# d3matrix1

## 题目简述

题目基于论文 [New Public-Key Cryptosystem Blueprints Using Matrix Products in $\mathbb F_p$](https://eprint.iacr.org/2023/1745)。论文研究的基础问题是：给定矩阵集合

$$
\mathbf A=\{A_0,\ldots,A_{k-1}\}
$$

以及由其中某个有序子集相乘得到的矩阵 $M$，能否恢复参与乘积的有序子集。论文据此提出两种矩阵公钥密码蓝图，并通过变换隐藏原始小矩阵及其乘积关系。

本题不要求直接恢复完整私钥，而是恢复公钥对应的每个原始矩阵之和。参数为

$$
p=2^{302}+307,\qquad k=140,\qquad n=10,
$$

且原始矩阵元素很小，满足 $a_{i,j}\in\{0,1,2\}$。由于 $k>n^2$，把每个 $n\times n$ 矩阵展平后必然存在高维线性关系；小元素约束又允许通过正交格和格规约恢复被打乱的矩阵行。

## 解题过程

### 从展平矩阵构造关系格

把每个公开矩阵 $D_i$ 展平成长度 $n^2$ 的列向量，组成

$$
M_D=(\operatorname{vec}(D_1),\ldots,\operatorname{vec}(D_k))
\in\mathbb F_p^{n^2\times k}.
$$

齐次方程

$$
M_Dx=0
$$

的核维数至少为

$$
k-n^2=140-100=40.
$$

先把模 $p$ 的右核嵌入整数格并执行 LLL：

```sage
from sage.all import (
    GF,
    Matrix,
    ZZ,
    block_matrix,
    identity_matrix,
    vector,
    zero_matrix,
)

def right_kernel_lattice(matrix, modulus, balance=1):
    matrix = Matrix(GF(modulus), matrix)
    rows, cols = matrix.nrows(), matrix.ncols()
    left = matrix[:, :rows]
    right = matrix[:, rows:]
    relation = Matrix(ZZ, -left.inverse() * right)

    lattice = block_matrix([
        [identity_matrix(rows) * modulus, zero_matrix(rows, cols - rows)],
        [relation.transpose(), identity_matrix(cols - rows)],
    ])
    lattice[-1, -1] = balance
    return lattice.LLL()
```

### 利用小元素与平衡坐标恢复矩阵

原始元素属于 $\{0,1,2\}$，减去 1 后变为 $\{-1,0,1\}$，更适合短向量恢复。若 $X$ 是前一步得到的关系格基，则追加每行元素和作为新坐标：

$$
X'=(X,\operatorname{rowsum}(X)).
$$

对 $X'$ 再取正交格，并对最短的 $n^2+1$ 个方向执行 BKZ。最后一维绝对值为 1 的短向量对应一个被置乱的原始位置；乘回该符号并加 1，就能把中心化元素恢复到 $\{0,1,2\}$。

```sage
p = 2**302 + 307
k = 140
n = 10
alpha = 3

flattened = [matrix.list() for matrix in public_matrices]
public = Matrix(GF(p), n**2, k)
for column in range(k):
    for row in range(n**2):
        public[row, column] = int(flattened[column][row])

relations = right_kernel_lattice(public, p)[:k - n**2]
row_sums = vector(ZZ, [sum(map(int, row)) for row in relations])
extended = relations.transpose().stack(row_sums).transpose()

orthogonal = right_kernel_lattice(extended, p, balance=1)
short = orthogonal[:n**2 + 1].BKZ(block_size=30)

# 官方数据中有一个无关方向，删除后筛选最后坐标为 ±1 的向量。
short = short[:98].stack(short[99:])
shuffled_A = []
for row in short.rows():
    sign = row[-1]
    if abs(sign) != 1:
        continue
    candidate = [value * sign + 1 for value in row[:-1]]
    if all(0 <= value < alpha for value in candidate):
        shuffled_A.append(candidate)
```

被打乱的行顺序不会影响本题所需的各矩阵元素和。对恢复矩阵按列求和，得到 `Asumlist`，其字符串表示的 SHA-256 摘要作为 AES-ECB 密钥：

```python
import hashlib
from Crypto.Cipher import AES

Asumlist = list(sum(Matrix(ZZ, shuffled_A)))
key = hashlib.sha256(str(Asumlist).encode()).digest()
flag = AES.new(key, AES.MODE_ECB).decrypt(flag_ciphertext)
```

官方题解验证结果为：

```text
d3ctf{OrTh0g0n@L_L@Tt1Ce_iS_wOnDe2fU1}
```

论文链接仍予保留，作为密码蓝图的完整定义和背景；但本题所需的攻击条件——$k>n^2$、矩阵元素很小、线性关系与正交格可恢复被打乱行——已经写入本文，无需依赖论文才能理解解法。

## 方法总结

- 核心技巧：把过完备的矩阵集合展平，先求模 $p$ 线性关系格，再求其正交格并用 LLL/BKZ 恢复小元素结构。
- 识别信号：矩阵数量超过向量空间维数，且秘密矩阵元素来自很小的集合时，公开变换往往无法隐藏短向量和正交关系。
- 复用要点：先把小元素中心化为对称区间；追加行和坐标可把常数平移纳入格关系；最终任务若只需要矩阵和，恢复出的行置乱不必额外消除。
