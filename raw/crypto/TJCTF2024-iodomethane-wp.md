# iodomethane

## 题目简述

题目实现了一个 $3\times3$ Hill 型密码。flag 中每个字符先映射为公开字符表中的小整数，随后每三个数构成列向量，与未知矩阵 $K$ 在大素数模

$$
p=15106021798142166691
$$

下相乘。密文和字符表均公开。目标是恢复解密矩阵 $D=K^{-1}$，再把所有密文向量映射回字符索引。

## 解题过程

解密矩阵每一行可以独立求解。设某一行是 $(a,b,c)$，密文三元组为 $(x_i,y_i,z_i)$，对应明文位置的字符索引为 $m_i$，则有

$$
ax_i+by_i+cz_i\equiv m_i\pmod p.
$$

flag 固定前缀 `tjctf{` 提供前六个字符，末尾 `}` 还提供一个已知字符。按位置模 3 分组后：

- 第 0 行已知前缀中的两个 `t`，只需枚举第三个字符；
- 第 1 行已知 `j`、`f` 和末尾 `}`，三条方程可直接确定；
- 第 2 行已知 `c`、`{`，再枚举第三个字符。

每次用三组明密文对求一个候选行，然后解密所有落在该行的位置。如果出现负值意义外或大于字符表长度的索引，就立即淘汰。

```python
from itertools import product

def inv_matrix_3(rows, mod):
    aug = [[v % mod for v in row] + [int(i == j) for j in range(3)]
           for i, row in enumerate(rows)]
    for col in range(3):
        pivot = next(r for r in range(col, 3) if aug[r][col])
        aug[col], aug[pivot] = aug[pivot], aug[col]
        scale = pow(aug[col][col], -1, mod)
        aug[col] = [(v * scale) % mod for v in aug[col]]
        for r in range(3):
            if r == col:
                continue
            factor = aug[r][col]
            aug[r] = [(x - factor * y) % mod
                      for x, y in zip(aug[r], aug[col])]
    return [row[3:] for row in aug]

def solve_row(cipher_rows, known_indexes, mod):
    inverse = inv_matrix_3(cipher_rows, mod)
    return [sum(inverse[r][c] * known_indexes[c] for c in range(3)) % mod
            for r in range(3)]
```

官方脚本得到 1 个第 0 行候选、1 个第 1 行候选和 75 个第 2 行候选；把三行组合并用“所有输出必须是合法字符索引”过滤后，只剩一个完整可读结果：

```text
tjctf{aint_no_hillllll_55e4$S56a356^#@!$}
```

## 方法总结

- Hill 密码的矩阵行彼此独立，可以利用零散已知明文逐行恢复，不必一次猜出完整矩阵。
- 大模数不能弥补小字符表带来的强约束。错误矩阵几乎总会产生远超字符表范围的数值，因此合法索引是有效判据。
- 固定 flag 前后缀不仅用于最终识别，也可以直接充当线性方程右端；应先按位置模分组，最大化每个已知字符的价值。
