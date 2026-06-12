# λm

## 题目简述

题目把一个自定义 VM 和字节运算全部编译成 lambda calculus 表达式，使用 Scott encoding 表示布尔值、pair、数组和 8-bit 整数。输入格式要求为 `LilacCTF{...}`，内部校验 flag 去掉包裹后的 32 字节。

VM 的可读语义是：把 32 字节输入拆成两个 16 字节数组 `N` 和 `D`，在 GF(2^8) 上构造递推基底 `B_i(x)`，然后把输入解释为有理函数 `N(x) / D(x)`，要求它在 40 个给定点上的取值全部等于内置 `Values_Y`。

## 解题过程

### VM 语义

`chall_src.py` 定义了 8-bit 算术和 GF(2^8) 运算，包括：

```python
b8_add
b8_sub
b8_mul
b8_div
b8_mod
b8_gfmul
b8_eq
```

`vmcode_src.txt` 展示了 VM 的高层逻辑：把 32 字节输入拆成两个 16 字节系数数组 `N` 和 `D`，对 40 个不同的 `x` 生成递归基底：

```text
B_0 = 1
B_1 = x
B_i = x * B_{i-1} ^ (x + 1) * B_{i-2}
```

然后计算：

$$
Y(x) = \frac{\sum_i N_i B_i(x)}{\sum_i D_i B_i(x)}
$$

并与内置的 40 个 `Values_Y` 比较。

### 还原为线性代数

虽然最终表达式被 lambda calculus 混淆，但校验本质是 GF(2^8) 上的有理函数插值。对每个样本点 `(x_k, y_k)`，有：

$$
y_k \cdot D(x_k) - N(x_k) = 0
$$

把 `N_0..N_15, D_0..D_15` 作为 32 个未知数，就得到 GF(2^8) 上的齐次线性方程组。`decrypt.sage` 中的求解流程是：

```sage
row = []
for i in range(16):
    row.append(B[i])
for j in range(16):
    row.append(y_val * B[j])

M = Matrix(F, matrix_rows)
K = M.right_kernel()
```

齐次方程的解只差一个非零标量，因此枚举 `1..255` 的标量，找到全部系数都落在可打印 ASCII 范围的解，即可恢复 32 字节内部 flag。

### 验证

`encrypt.sage` 中对应的明文内部串为：

```text
R4t10n4l_Cha05_w17h_R3cur510n_VM
```

拼回题目要求格式即：

```text
LilacCTF{R4t10n4l_Cha05_w17h_R3cur510n_VM}
```

将该字符串输入 `chall.py`，表达式最终返回真并输出 `Correct!`。

## 方法总结

- 核心技巧：不要直接化简巨型 lambda 表达式，而是从源 VM 恢复高层数学语义，把校验转成 GF(2^8) 线性代数。
- 识别信号：VM 中大量 8-bit 运算、数组、基底递推和多点输出比较，往往是在隐藏插值或线性方程问题。
- 复用要点：遇到 lambda/函数式混淆时，优先识别编码方式和高层数据流；一旦还原出真实数学关系，求解通常比继续展开表达式简单得多。
