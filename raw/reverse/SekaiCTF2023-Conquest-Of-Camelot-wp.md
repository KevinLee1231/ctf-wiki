# Conquest of Camelot

## 题目简述

题目提供一个带符号和调试信息的 64 位 OCaml ELF。程序要求输入长度恰为 36 的 flag，把各字符 ASCII 值送入三层网络，并将 29 维输出与内置目标向量比较；最大误差不超过 $10^{-6}$ 才会显示 `Quest Completed!`。

虽然表面上像神经网络逆向，但三层全是 `Linear`，没有任何非线性激活。整个检查器实际上只是一个仿射变换，可以合并成普通线性方程组。

## 解题过程

根据符号名和 `camlDune__exe__Camelot__entry` 可恢复主要结构。随机数种子固定为 `0x1337`，权重和偏置均由 OCaml `Random.int 200` 生成：

```ocaml
weight.(i).(j) <-
  float_of_int (Random.int 200 - 100) /. 100.0;
bias.(i) <-
  float_of_int (Random.int 200 - 100) /. 100.0
```

三个线性层的形状为：

```text
36 -> 512 -> 137 -> 29
```

可用一个小型 OCaml 生成器，以相同种子和调用顺序输出：

```text
W1, b1, W2, b2, W3, b3
```

不要改用其他语言的同名随机函数，因为不同运行时的 PRNG 和取样过程并不等价。题目中的每个随机系数只有两位小数，转成 `Fraction` 后可进行精确有理数运算，避免三次矩阵乘法带来的浮点误差。

设输入字符列向量为 $x$。前向过程为：

$$
y = W_3\left(W_2\left(W_1x+b_1\right)+b_2\right)+b_3.
$$

展开可得：

$$
y = W_3W_2W_1x + W_3W_2b_1 + W_3b_2 + b_3.
$$

因此令：

$$
A=W_3W_2W_1,
$$

$$
c=y-W_3W_2b_1-W_3b_2-b_3,
$$

问题就变为 $Ax=c$。程序内置的目标向量有 29 项，而 $x$ 有 36 项；再加入 flag 格式和可打印字符约束即可求解。将有理系数统一放大为整数后交给 Z3：

```python
from z3 import BitVec, Solver

x = [BitVec(f"x{i}", 8) for i in range(36)]
s = Solver()

prefix = b"SEKAI{"
for i, ch in enumerate(prefix):
    s.add(x[i] == ch)
s.add(x[35] == ord("}"))

for i in range(6, 35):
    s.add(x[i] >= 32, x[i] <= 126)

# A_int 与 c_int 由 Fraction 矩阵统一放大得到。
for i in range(29):
    s.add(
        sum(A_int[i][j] * x[j] for j in range(36))
        == c_int[i]
    )

assert s.check().r == 1
model = s.model()
flag = bytes(model[ch].as_long() for ch in x)
print(flag.decode())
```

解得：

```text
SEKAI{n3ur4l_N3T_313c7R0n_C0mbO_uwu}
```

把该字符串输入原程序，可得到：

```text
Quest Completed!
```

## 方法总结

判断模型是否真的“非线性”比直接上机器学习工具更重要。连续仿射层仍是一个仿射层，合并后只需解 $Ax=c$。本题另一个容易出错之处是 OCaml 固定种子的随机序列与浮点精度：用原运行时恢复参数，再把两位小数系数转换成有理数，能够同时保证权重顺序和方程精度。
