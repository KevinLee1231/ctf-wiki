# Hashbrown

## 题目简述

WelcomeCTF2021 的 Hashbrown 把 flag 切成 10 个 32 位整数 $v_1,\ldots,v_{10}$，并给出随机系数 $C_i$ 与目标值 $H$，满足

$$
\sum_{i=1}^{10} C_i v_i \equiv H \pmod{2^{512}},
\qquad 0\le v_i<2^{32}.
$$

单个同余方程看似欠定，但每个未知量都很小，可以用 LLL 找到对应的短向量。

## 解题过程

官方脚本构造一个 12 维整数格。前 10 行在各自坐标放置单位向量，并把 $C_i$ 放入第 11 列；第 11 行放置目标 $H$，同时在最后一列放置 $2^{32}$；第 12 行在第 11 列放置模数 $2^{512}$：

```python
matrix = [[0 for _ in range(12)] for _ in range(12)]

for i in range(10):
    matrix[i][i] = 1
    matrix[i][10] = coeff[i]

matrix[10][10] = H
matrix[10][11] = 1 << 32
matrix[11][10] = 1 << 512
```

格中若取前十行系数为 $-v_i$、目标行系数为 $1$，再用模数行消去 $H-\sum C_i v_i$，第 11 坐标就会变成 0。所得向量的前十个坐标绝对值不超过 $2^{32}$，最后一个坐标也是 $2^{32}$，明显比带有未消去 512 位同余残差的向量短。

因此执行 LLL 后，最短行会给出目标组合：

```python
reduced = Matrix(ZZ, matrix).LLL()
row = reduced.row(0)

flag = b""
for i in range(10):
    flag += int(-row[i]).to_bytes(4, "big")
print(flag)
```

最后一列的 $2^{32}$ 用于约束目标行的系数尽量取 $\pm1$，避免 LLL 返回目标值的较大倍数。恢复的十个整数按大端序各转为 4 字节并拼接，得到：

```text
greyhats{L@t7ic3_br3ak5_y0ur_ha5h_m8!!!}
```

## 方法总结

这是一类典型的有界模线性方程。构造格时既要编码同余模数，也要让目标 $H$ 以受控系数进入组合；LLL 才会把“小未知量且同余残差为零”的解识别为短向量。还原后应重新计算同余式并检查每个 $v_i$ 的 32 位范围，再进行字节拼接。
