# week2linearEquation

## 题目简述

附件 `output.txt` 给出 32 个未知数 $x_0,\ldots,x_{31}$ 的 32 条一次方程。每个未知数都是随机生成的 32 位非负整数，flag 的生成规则由题目源码注明为：先计算 $S=\sum_{i=0}^{31}x_i$，再取十进制字符串 `str(S)` 的 MD5，即 `0xGame{md5(str(S))}`。

## 解题过程

每行都可以写成

$$
\sum_{j=0}^{31}a_{ij}x_j=b_i,
$$

把 32 行系数构成矩阵 $A$，等号右侧构成列向量 $b$，问题就是在整数域中求解 $Ax=b$。附件中的矩阵满秩，因此解唯一。原 PDF 将全部长方程手工粘进 Sage 的 `solve`；更稳妥的做法是直接解析 `output.txt`，这样既不会在复制时漏项，也能保留完整可复现性。

```python
import hashlib
import re

from sympy import Matrix

A = []
b = []

with open("output.txt", "r", encoding="utf-8") as f:
    for raw_line in f:
        line = raw_line.strip().rstrip(",")
        if not line:
            continue

        left, right = line.split("==", 1)
        coeffs = [
            int(value)
            for value in re.findall(r"x\d+\s*\*\s*(\d+)", left)
        ]
        assert len(coeffs) == 32
        A.append(coeffs)
        b.append(int(right.strip()))

assert len(A) == len(b) == 32

solution = Matrix(A).LUsolve(Matrix(b))
assert all(value.q == 1 for value in solution)
x = [int(value) for value in solution]

total = sum(x)
digest = hashlib.md5(str(total).encode()).hexdigest()

print("sum(x) =", total)
print(f"0xGame{{{digest}}}")
```

运行结果为：

```text
sum(x) = 68580473339
0xGame{d23caedc66301966094bf8968610f61f}
```

## 方法总结

本题的关键是把冗长文本识别为标准的方阵线性系统，而不是逐个猜测未知数。应从原始输出自动提取系数和常数项，使用精确有理数运算求解，再确认 32 个解均为整数。最后的 MD5 输入是 $\sum x_i$ 的十进制文本，不是整数的二进制表示，也不能额外带换行符。
