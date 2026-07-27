# d3matrix2

## 题目简述

题目攻击论文 [New Public-Key Cryptosystem Blueprints Using Matrix Products in $\mathbb F_p$](https://eprint.iacr.org/2023/1745) 提出的第一种矩阵公钥密码蓝图。论文的基础困难问题是：已知一组矩阵和某个有序子集的乘积，恢复参与乘积的矩阵及其顺序；蓝图再通过秘密变换隐藏原始矩阵。

本题公开 99 个 $24\times24$ 矩阵 $D_i$ 和乘积密文 $c$。实例中的矩阵元素非负且较小，导致“继续乘入矩阵通常使迹增大”这一非通用但对当前数据成立的泄漏。解法利用迹在相似变换和循环换位下不变，从乘积两端逐个剥离矩阵，恢复完整排列。

## 解题过程

### 迹泄漏

矩阵迹满足

$$
\operatorname{tr}(ABC)
=\operatorname{tr}(BCA)
=\operatorname{tr}(CAB),
$$

相似矩阵也具有相同的迹。因此秘密共轭或置换变换不会消除实例中的迹大小关系。

假设当前密文可写为

$$
c=D_i\,c'\,D_j.
$$

左乘 $D_i^{-1}$ 后，

$$
\operatorname{tr}(D_i^{-1}c)
=\operatorname{tr}(c'D_j)
<\operatorname{tr}(D_i c'D_j)
=\operatorname{tr}(c).
$$

右端矩阵 $D_j$ 也会满足同类关系，因为

$$
\begin{aligned}
\operatorname{tr}(D_j^{-1}c)
&=\operatorname{tr}(D_j^{-1}D_i c'D_j)\\
&=\operatorname{tr}(D_jD_j^{-1}D_i c')\\
&=\operatorname{tr}(D_i c')
<\operatorname{tr}(D_i c'D_j).
\end{aligned}
$$

这两组公式原本是图片中的纯公式，现已转写为 Markdown 数学表达式，不再保留公式截图。

### 从两端剥离排列

每轮遍历尚未使用的 $D_i$，测试

$$
\operatorname{tr}(D_i^{-1}c)<\operatorname{tr}(c).
$$

通常只会得到当前乘积的首、尾两个候选。再比较两个剥离顺序：

$$
\operatorname{tr}(D_i^{-1}cD_j^{-1})
\quad\text{与}\quad
\operatorname{tr}(D_j^{-1}cD_i^{-1}),
$$

即可判断谁在左端、谁在右端。确认后更新 `prefix`、`suffix` 和剩余密文，直到只剩一个矩阵。

```sage
prefix, suffix = [], []

while True:
    trace_c = c.trace()
    candidates = []

    for index, matrix in enumerate(public_matrices):
        if matrix is None:
            continue
        if (matrix**-1 * c).trace() < trace_c:
            candidates.append(index)

    if len(candidates) == 1:
        permutation = prefix + candidates + suffix
        break

    left, right = candidates
    trace_lr = (
        public_matrices[left]**-1
        * c
        * public_matrices[right]**-1
    ).trace()
    trace_rl = (
        public_matrices[right]**-1
        * c
        * public_matrices[left]**-1
    ).trace()

    if trace_lr < trace_c and trace_rl > trace_c:
        prefix.append(left)
        suffix.insert(0, right)
        c = (
            public_matrices[left]**-1
            * c
            * public_matrices[right]**-1
        )
    elif trace_rl < trace_c and trace_lr > trace_c:
        prefix.append(right)
        suffix.insert(0, left)
        c = (
            public_matrices[right]**-1
            * c
            * public_matrices[left]**-1
        )
    else:
        raise ValueError("trace heuristic is not unique")

    public_matrices[left] = None
    public_matrices[right] = None
```

得到排列 `permutation` 后，以其字符串表示的 SHA-256 摘要作为 AES-ECB 密钥：

```python
import hashlib
from Crypto.Cipher import AES

key = hashlib.sha256(str(permutation).encode()).digest()
flag = AES.new(key, AES.MODE_ECB).decrypt(flag_ciphertext)
```

官方材料没有给出脚本最终打印的 flag，因此不补写未经验证的明文。

这里保留论文 URL，供查看完整密码蓝图；但本题实际依赖的矩阵乘积模型、相似变换的迹不变量、实例特有的迹单调性和剥离算法均已在正文中说明。

## 方法总结

- 核心技巧：利用迹的循环不变性，从有序矩阵乘积的左右两端逐轮识别并剥离因子。
- 识别信号：方案依赖共轭或相似变换隐藏矩阵，但实例矩阵具有非负小元素、乘法后迹明显增大的统计规律时，应检查迹、行列式和特征多项式等不变量。
- 复用要点：迹随乘法增大不是一般定理，必须先在题目实例上验证；每轮同时出现两个候选时，需要比较双边剥离顺序，不能只按首次命中的下标决定排列。
