# DownUnderCTF 2023 Masked Squares Flag Checker Writeup

## 题目简述

校验器要求输入恰好 36 个字符，并把它们按行排成 $6\times6$ 方阵。程序内置 26 个经过游程编码的二值掩码；每个掩码选中若干格子，再要求这些格子的 ASCII 码之和等于对应常量。

每条检查都是线性等式，因此没有必要逐字符爆破。解压掩码后，把 36 个字符作为有界整数变量交给约束求解器即可。

## 解题过程

掩码数据使用带符号游程编码：

- 正数 $n$ 表示连续 $n$ 个 `1`，这些位置参与求和；
- 负数 $-n$ 表示连续 $n$ 个 `0`；
- `0` 表示当前掩码结束。

按行展开 36 个格子后，解码函数可以写成：

```python
def uncompress_mask(encoded):
    out = [[0] * 6 for _ in range(6)]
    x = y = 0
    for value in encoded:
        if value == 0:
            break
        bit = 1 if value > 0 else 0
        for _ in range(abs(value)):
            out[y][x] = bit
            x += 1
            if x == 6:
                x = 0
                y += 1
    return out
```

设第 $j$ 个字符的 ASCII 码为 $f_j$，第 $k$ 个掩码在该位置的取值为 $m_{k,j}\in\{0,1\}$，对应输出常量为 $s_k$，每条检查就是

$$
\sum_{j=0}^{35}m_{k,j}f_j=s_k.
$$

为排除无意义的整数解，将每个变量限制在可打印 ASCII 范围 $[32,126]$，并加入已知格式 `DUCTF{...}`。使用 OR-Tools CP-SAT 的核心代码如下：

```python
from ortools.sat.python import cp_model

model = cp_model.CpModel()
flag = [model.NewIntVar(32, 126, f"f{i}") for i in range(36)]

for i, value in enumerate(b"DUCTF{"):
    model.Add(flag[i] == value)
model.Add(flag[-1] == ord("}"))

for target, encoded_mask in zip(outputs, compressed_masks):
    mask = uncompress_mask(encoded_mask)
    selected = [
        flag[row * 6 + col]
        for row in range(6)
        for col in range(6)
        if mask[row][col]
    ]
    model.Add(sum(selected) == target)

solver = cp_model.CpSolver()
assert solver.Solve(model) in (cp_model.FEASIBLE, cp_model.OPTIMAL)
print(bytes(solver.Value(ch) for ch in flag).decode())
```

约束系统给出的完整输入为：

```text
DUCTF{ezzzpzzz_07bcda7bfe81faf43caa}
```

源码中的 `scanf("%36s", buf)` 还会在 36 字节缓冲区末尾多写一个终止零字节，但这个边界问题并不是取得 flag 所必需的路径。

## 方法总结

看到大量“按掩码选择字符并求和”的检查时，应先判断其代数结构，而不是照着控制流逐项猜测。本题的游程编码只是表示层；解码后，主体是带字符范围和已知前后缀的整数线性约束系统，直接使用 CP-SAT 或 SMT 求解最稳妥。
