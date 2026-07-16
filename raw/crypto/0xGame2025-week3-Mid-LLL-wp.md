# Mid_LLL

## 题目简述

题目把 flag 的每个字节作为向量 `msg`，放到一个方阵的第一行，其余元素均为 $[0,2^{128}]$ 内的随机整数；随后再生成同维随机矩阵 `noise`，只公开整数矩阵乘积 `noise * matrix`。目标是从输出行格中恢复被大整数线性组合隐藏的短字节向量。

## 解题过程

记原矩阵为 $M$、噪声矩阵为 $N$、公开矩阵为 $O=NM$。即使 $N$ 通常不是整系数可逆矩阵，也有：

$$
\operatorname{adj}(N)O=\det(N)M.
$$

因此 $O$ 的行格中包含原矩阵每一行的某个整数倍，特别是 $k\cdot msg$。`msg` 的坐标只是 ASCII 字节，而其它原始行的坐标约为 128 位随机数，所以该方向在格中明显更短。对公开矩阵做 LLL 后，首个短向量通常就是带符号的 $k\cdot msg$。

```sage
from sage.all import Matrix, ZZ, gcd

output = [...]  # 题目给出的完整整数矩阵
reduced = Matrix(ZZ, output).LLL()
vector = reduced[0]

scale = abs(gcd(list(vector)))
message = bytes(abs(int(x)) // scale for x in vector)
print(message.decode())
```

对坐标取最大公约数可以消去公共倍数 $k$，取绝对值则处理 LLL 返回相反方向的情况。仓库中的 `output.txt` 运行后得到：

```text
0xGame{B3g1nn3r_t0_1e4rn_L4tt1c3s}
```

## 方法总结

- 矩阵乘法隐藏的是行向量的整数线性组合，可以从“输出行格是否仍含原短方向”入手。
- 非幺模变换未必保持原格完全相同，但伴随矩阵关系保证输出格中包含原行向量的公共整数倍。
- LLL 后不要直接把大整数当字符；先通过坐标 GCD 消去比例，再处理符号并按字节解码。
