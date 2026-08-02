# dotdotdotv2

## 题目简述

生成器把一段公开的长前缀与 flag 拼接，逐字符转成 8 位 ASCII 二进制，再切成 64 位行向量 $x$。随机生成的 $64\times64$ 整数矩阵 $K$ 被重复用于所有分组，输出每行的普通整数点积 $y=xK$，没有取模。题目提示 `np.dot`，决定性弱点是固定线性变换和大量已知明文。

## 解题过程

普通 ASCII 的最高位为 0，所以每个 64 位分组中索引为 $0,8,16,\ldots,56$ 的 8 位恒为 0，真正未知的只有其余 56 位。公开前缀足够构造至少 56 个已知分组；删去恒零列后，可得到一个 $56\times56$ 的已知明文矩阵 $P$。

对输出的每一列 $j$，取前 56 个密文值组成向量 $c_j$，解线性方程

$$P k_j=c_j$$

即可恢复该列在 56 个有效输入位上的系数。恢复完整的有效矩阵后，再对每个密文分组求解输入位。由于运算均为整数且真实解是 0/1，浮点求解结果四舍五入即可；更严格的实现可使用有理数消元。

```python
import numpy as np

PREFIX = (
    "In cybersecurity, a CTF (Capture The Flag) challenge is a competitive, "
    "gamified event where participants, either individually or in teams, are "
    "tasked with finding and exploiting vulnerabilities in systems to capture "
    "hidden information known as flags. These flags are typically used to score "
    "points. CTFs test skills in areas like cryptography, web security, reverse "
    "engineering, and forensics, offering an exciting way to learn, practice, "
    "and showcase cybersecurity expertise.  This flag is for you: tjctf{"
)

known_blocks = []
for start in range(0, len(PREFIX), 8):
    block = PREFIX[start:start + 8]
    if len(block) == 8:
        bits = "".join(f"{ord(ch):08b}" for ch in block)
        known_blocks.append([int(bit) for bit in bits])

with open("encoded.txt", "r", encoding="utf-8") as f:
    encoded = [list(map(int, line.split())) for line in f]

# 每个 ASCII 字节的最高位恒为 0，删除这些无效位置。
P = np.array([
    [bit for i, bit in enumerate(row) if i % 8 != 0]
    for row in known_blocks[:56]
], dtype=float)

columns = []
for output_column in range(64):
    values = np.array([row[output_column] for row in encoded[:56]], dtype=float)
    columns.append(np.rint(np.linalg.solve(P, values)).astype(int))

# 取 56 个线性无关的输出坐标，逐块恢复 56 个有效输入位。
effective = np.array(columns[:56], dtype=float)
text = []
for row in encoded:
    bits = np.rint(np.linalg.solve(effective, row[:56])).astype(int)
    for start in range(0, 56, 7):
        text.append(chr(int("0" + "".join(map(str, bits[start:start + 7])), 2)))

print("".join(text).rstrip("\x00"))
```

附件数据的输出末尾包含：

```text
tjctf{us3fu289312953}
```

## 方法总结

- 核心技巧：用大量已知明文恢复被重复使用的线性变换矩阵。
- 识别信号：`np.dot`、固定矩阵、无取模运算以及长公开前缀共同把问题变成线性方程组。
- 复用要点：先删除确定为零的输入维度，避免矩阵奇异；数值求解后要检查结果是否接近 0/1，并用重新乘回矩阵验证。
