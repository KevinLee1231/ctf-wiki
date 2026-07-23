# Linear Algebra

## 题目简述

程序要求输入去掉 `UMDCTF-{}` 后的 18 个字符。前两阶段固定若干下划线和数字，最终阶段把字符分成两个 3×3 方阵，并约束各行、各列及一个额外组合的 ASCII 和。

## 解题过程

把 18 个字符建模为可打印 ASCII 整数。源码中的固定条件与和式可以直接交给 Z3：

```python
from z3 import IntVector, Solver, sat

x = IntVector("x", 18)
s = Solver()
for value in x:
    s.add(value >= 0x20, value <= 0x7e)

fixed = {
    1: "_", 6: "_", 11: "_", 16: "_",
    3: "0", 5: "3", 13: "0",
}
for index, char in fixed.items():
    s.add(x[index] == ord(char))

equations = [
    ([0, 1, 2], 0xf4), ([3, 4, 5], 0xd9),
    ([6, 7, 8], 0x10d), ([0, 3, 6], 0xd8),
    ([1, 4, 7], 0x122), ([2, 5, 8], 0xe0),
    ([9, 10, 11], 0x11b), ([12, 13, 14], 0x102),
    ([15, 16, 17], 0x128), ([9, 12, 15], 0x14c),
    ([10, 13, 16], 0xd7), ([11, 14, 17], 0x122),
    ([9, 15], 0xe8),
]
for indexes, total in equations:
    s.add(sum(x[index] for index in indexes) == total)

assert s.check() == sat
model = s.model()
answer = "".join(chr(model[value].as_long()) for value in x)
print(answer)
```

解得程序实际检查的 18 字符文本：

```text
I_L0v3_MatH_d0nt_U
```

完整 flag 为：

```text
UMDCTF-{I_L0v3_MatH_d0nt_U}
```

该字符串的 SHA-256 与 README 中的记录一致；程序没有检查更多后缀字符。

## 方法总结

这里的“线性代数”就是有限个整数线性方程。固定字符条件能消除方程组的自由度；把字符限制在可打印 ASCII 后，用符号求解比手工消元更直接，也便于逐条核对源码约束。
