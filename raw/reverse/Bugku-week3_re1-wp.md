# week3_re1

## 题目简述

题目是 64 位 Windows PE 线性方程组逆向。程序读取 42 字符输入，将缓冲区按 little-endian 解析成 12 个 `uint32`，再代入 12 个线性方程。每个方程形如：

```text
sum(coeff[i][j] * var[j]) + 0x2022 == expected[i]  (mod 2^64)
```

满足全部 12 个方程后输出正确。核心解法是从反编译中提取 12x12 系数矩阵和 12 个 64 bit 期望值，用 Z3 求解。

## 解题过程

### 关键观察

主函数流程：

```text
scanf("%s", buf)
strlen(buf) == 0x2a
buf -> 12 个 uint32 局部变量
计算 12 个 uint64 方程
memcmp(computed, DAT_00404020, 0x60)
```

由于输入只有 42 字节，最后一个 `uint32` 只有低 16 位来自输入，高 16 位为字符串终止符后的零；另一个越界参与计算的局部变量由 `memset` 初始化为 0。这两个约束能显著收紧 Z3 搜索。

### 求解步骤

用 64 bit `BitVec` 建模，保持乘法和加法的模 `2^64` 语义：

```python
from z3 import *

v = [BitVec(f"v{i}", 64) for i in range(12)]
s = Solver()

s.add(v[0] == 0)                    # memset 置零的额外变量
s.add(Extract(31, 16, v[11]) == 0)  # 42 字节输入后的高 16 位为 0
for x in v:
    s.add(Extract(63, 32, x) == 0)

for i in range(12):
    expr = BitVecVal(0x2022, 64)
    for j in range(12):
        c = coeffs[i][j]
        expr += BitVecVal(c % (1 << 64), 64) * v[j]
    s.add(expr == BitVecVal(expected[i], 64))

assert s.check() == sat
m = s.model()
raw = b"".join(m[x].as_long().to_bytes(4, "little") for x in v)
print(raw[4:46].decode())
```

求解结果：

```text
flag{1853dd27-8720-48c8-a725-1fe52b8b25e7}
```

## 方法总结

- 核心技巧：将线性校验直接转为 SMT/线性方程约束求解。
- 识别信号：大量 `coeff * local_var` 累加后与常量表比较，且变量来自输入分组，通常适合矩阵或 Z3。
- 复用要点：局部变量布局会影响约束，尤其是缓冲区末尾未满 4 字节和 `memset` 产生的零变量；忽略这些会让模型不唯一或无解。
