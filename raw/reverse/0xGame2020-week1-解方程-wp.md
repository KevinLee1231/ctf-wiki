# week1解方程

## 题目简述

程序读取 28 个字符，把它们按每 7 字节一组分成 4 组。每组都乘以同一个 $7\times7$ 系数矩阵，再与 `check` 数组中的 7 个整数比较。所有约束都是普通整数线性方程，可以直接交给 Z3 求解。

## 解题过程

源码中 28 条展开式看似很长，实际只是同一矩阵作用于四个连续字符块。将重复结构提炼为系数矩阵，既保留全部约束，也能避免手工抄写 28 条表达式时出错：

```python
from z3 import Int, Solver, Sum, sat

coefficients = [
    [12, 53, 6, 34, 58, 36, 1],
    [83, 85, 12, 73, 27, 96, 52],
    [78, 53, 24, 36, 86, 25, 46],
    [39, 78, 52, 9, 62, 37, 84],
    [23, 6, 14, 74, 48, 12, 83],
    [27, 85, 92, 42, 48, 15, 72],
    [4, 6, 3, 67, 0, 26, 68],
]

check = [
    20933, 41536, 33625, 38288, 27097, 40649, 18710,
    18617, 41846, 36080, 35205, 28888, 36346, 18809,
    20789, 39340, 31803, 33629, 24418, 34136, 16504,
    19638, 40515, 36396, 37839, 28986, 41711, 18605,
]

flag = [Int(f"x{i}") for i in range(28)]
solver = Solver()

for char in flag:
    solver.add(char >= 0x20, char <= 0x7E)

for block in range(4):
    for row in range(7):
        expression = Sum(
            coefficients[row][column] * flag[block * 7 + column]
            for column in range(7)
        )
        solver.add(expression == check[block * 7 + row])

assert solver.check() == sat
model = solver.model()
print("".join(chr(model[char].as_long()) for char in flag))
```

输出为：

```text
0xgame{z3_ls_v3r7_ve_awful!}
```

这里使用 `Int` 而不是 8 位 `BitVec`。C 语言中的 `unsigned char` 参与乘法前会提升为 `int`，原程序比较的是未截断的整数和；若用 8 位位向量，Z3 会按模 256 溢出，建模语义并不等价。

## 方法总结

- 核心技巧：识别重复的线性变换，把展开式压缩成矩阵约束交给 Z3。
- 识别信号：输入字符只参与常数乘法和加法，最终逐项与常量数组比较。
- 复用要点：Z3 类型要匹配程序的整数提升和溢出语义；重复约束优先抽成矩阵，减少漏项和下标错误。
