# Elgyem's Password

## 题目简述

题目给出一个全连接神经网络的精确权重、偏置以及某个未知输入对应的输出。网络各层之间没有 ReLU、Sigmoid 等非线性激活，因此整个模型只是一个仿射变换。目标是在“输入是 22 个可打印 ASCII 字符且符合 flag 前缀”的约束下恢复原输入。

## 解题过程

每一层都具有形式：

$$
\mathbf y_i=W_i\mathbf y_{i-1}+\mathbf b_i.
$$

连续仿射层可合并。例如三层网络展开后为：

$$
\mathbf y
=W_3W_2W_1\mathbf x
+W_3W_2\mathbf b_1
+W_3\mathbf b_2
+\mathbf b_3.
$$

因此全部六层可约化成 $\mathbf y=W\mathbf x+\mathbf b$。矩阵不是方阵，且组合映射存在非零核，直接求逆或伪逆不能保证返回唯一、合法的字符串；缺少离散字符约束时会得到许多实数解。

官方求解脚本保留原网络结构，用 Z3 为 22 个输入字节建立整数变量，并按 `model.txt` 中的有理数权重逐层计算。约束包括：

```python
for x in input_bytes:
    solver.add(x > 32, x < 128)

for i, value in enumerate(b"UMDCTF{"):
    solver.add(input_bytes[i] == value)

for i, expected in enumerate(outputs):
    solver.add(final_layer[i] == expected)
```

权重以诸如 `-713/1000` 的精确有理数保存，必须用 `RealVal` 或整数缩放保持精度；若改用普通浮点数，舍入误差会使等式约束不再严格成立。

求解模型并按索引将整数转为字符，得到：

```text
UMDCTF{n3uR4Ln37w0rk5}
```

实践中可先把网络合并成单个 $37\times22$ 仿射层，再交给 Z3，显著减少中间符号表达式和内存消耗。

## 方法总结

- 核心技巧：没有激活函数的深层网络并不增加表达能力，可以精确折叠为一个仿射变换。
- 伪逆只给出某个实数解；flag 格式和可打印 ASCII 是选出离散原像的必要约束。
- 精确权重应保留为有理数，不能在符号求解前转换成浮点近似。
- 先做矩阵合并能避免为大量隐藏层节点建立冗余约束。
