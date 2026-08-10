# 猫头鹰是不是猫

## 题目简述

程序先打印猫和猫头鹰的字符画，随后把 64 字节输入依次乘以两个 $64\times64$ 常数矩阵，并将结果与 64 个 DWORD 比较。字符画只是干扰；校验本质是两次可逆线性变换，需要从二进制中提取结果向量和两个矩阵，再求逆恢复输入。

## 解题过程

设输入行向量为 $x$，两个矩阵依次为 $A$、$B$，比较向量为 $y$。程序的计算关系可写为：

$$
y=xAB
$$

因此：

$$
x=yB^{-1}A^{-1}
$$

二进制把矩阵元素放大 10 倍后以 DWORD 保存，使用时再除以 10。原题映射到基址 0 后，结果数组位于 `0x4040`，矩阵 `A` 位于 `0x4140`，矩阵 `B` 位于 `0x8140`。可以用 angr 读取常量，再交给 NumPy：

```python
import angr
import archinfo
import numpy as np

project = angr.Project(
    "./out",
    auto_load_libs=False,
    main_opts={"base_addr": 0},
)
state = project.factory.blank_state()

def read_u32(address):
    value = state.memory.load(
        address,
        4,
        endness=archinfo.Endness.LE,
    )
    return state.solver.eval(value)

result = np.array([read_u32(0x4040 + 4 * i) for i in range(64)])
matrix_a = np.array(
    [read_u32(0x4140 + 4 * i) / 10 for i in range(64 * 64)]
).reshape(64, 64)
matrix_b = np.array(
    [read_u32(0x8140 + 4 * i) / 10 for i in range(64 * 64)]
).reshape(64, 64)

plain = result @ np.linalg.inv(matrix_b) @ np.linalg.inv(matrix_a)
flag = "".join(chr(round(value)) for value in plain)
print(flag)
```

浮点求逆会产生极小误差，所以输出前使用 `round`。公开复现记录给出的最终输入为：

```text
hgame{100011100000110000100000000110001010110000100010011001111}
```

矩阵维度、运算次序与最终值通过 [Second_BC 的逆向记录](https://secondbc.github.io/SecondBC/2022/02/22/Hgame2022-ReverseWriteUp/) 补充核对；正文已完整说明两轮矩阵变换与提取地址。

## 方法总结

遇到大规模嵌套乘加时，应先判断它是否是矩阵乘法，而不是逐条翻译循环。恢复公式后，最重要的是确认向量是左乘还是右乘，以及逆矩阵的相反顺序。字符画与题名不构成解题依据，真正的决定性信息是两个常数矩阵和比较向量。
