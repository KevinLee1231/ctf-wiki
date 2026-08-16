# HackINI2024 matrix

## 题目简述

题目给出一段长度为完全平方数的长文本。文本应被视为一个方阵，真正的信息被藏在矩阵主对角线上；目标是确定矩阵边长并按正确步长提取字符。

## 解题过程

设文本总长度为 $N$。检查可知 $N=217^2$，因此可以将其按行填入 $217\times217$ 的方阵。主对角线元素在线性数组中的下标依次为：

$$
0,\ 217+1,\ 2(217+1),\ldots
$$

也就是每次跨过 $218$ 个字符。无需真的构造二维列表，直接切片即可：

```python
import math
import re

with open("matrix.txt", "r", encoding="utf-8") as f:
    text = f.read().strip()

side = math.isqrt(len(text))
assert side * side == len(text)

diagonal = text[::side + 1]
flag = re.search(r"shellmates\{[^}]+\}", diagonal).group(0)
print(diagonal)
print(flag)
```

对角线文本以如下内容开头：

```text
finally_the_golden_letters_shellmates{YOu_vE_eaRnEd_w1$d0M_4ND_MaTH$Ss}_no_needy...
```

因此 flag 为：

```text
shellmates{YOu_vE_eaRnEd_w1$d0M_4ND_MaTH$Ss}
```

## 方法总结

方阵的主对角线在线性存储中的固定步长是“边长加一”。面对超长规则文本时，应先检查长度是否为完全平方数，再测试主对角线、次对角线、行列转置等常见空间布局。使用 `math.isqrt()` 可以避免浮点开方的精度问题，最后再用 flag 格式从含有填充文字的对角线结果中截取答案。
