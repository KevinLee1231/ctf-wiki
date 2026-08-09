# zzz

## 题目简述

程序对 30 字节输入施加大量加减、异或、按位与或和乘法约束。逐项手推容易出错，题名中的三个 `z` 提示使用 Z3 求解。

## 解题过程

为每个字符创建 8 位位向量，固定 flag 外壳，并逐条翻译 `check` 函数：

```python
from z3 import *

secret = [BitVec(f"s{i}", 8) for i in range(30)]
solver = Solver()
solver.add(secret[0] == ord('n'), secret[1] == ord('0'))
solver.add(secret[2] == ord('0'), secret[3] == ord('b'))
solver.add(secret[4] == ord('z'), secret[5] == ord('{'))
solver.add(secret[29] == ord('}'))

solver.add((secret[3] | secret[6]) == 122)
solver.add((secret[3] & secret[6]) == 66)
solver.add(secret[6] + secret[7] + secret[8] == 302)
solver.add(secret[6] * secret[7] - secret[8] == 10890)
# 其余条件继续按源码逐条加入
```

对正文字符补充可打印范围约束后求解模型，并按下标输出：

```text
n00bz{ZzZ_zZZ_zZz_ZZz_zzZ_Zzz}
```

## 方法总结

位运算必须用固定宽度 `BitVec` 建模，不能随意换成无界整数；同时应把源码中的全部条件和字符范围写入求解器，避免得到虽满足部分方程但不可提交的模型。
