# xorrrrrrrrr

## 题目简述

源码从同一篇文章中随机截取 100 个与 flag 等长的片段 $A[o_i:o_i+L]$，分别与同一个 flag $F$ 异或：

$$
C_i[j]=F[j]\oplus A[o_i+j].
$$

`result.log` 保存 100 个 Python 字节串。目标是在不知道文章和各个偏移量的情况下，利用重复明文片段恢复 flag。

## 解题过程

已知 flag 以 `moectf{` 开头，因此每条密文的前七字节都能泄露对应文章片段：

$$
C_i[0:7]\oplus\texttt{b"moectf{"}=A[o_i:o_i+7].
$$

用 `ast.literal_eval` 读取日志，比较这些片段，可以发现第 33 条与第 28 条来自文章中相邻的位置：第 28 条的起点比第 33 条晚一字节。于是对 $j\ge1$ 有

$$
\begin{aligned}
C_{33}[j] &= F[j]\oplus A[o+j],\\
C_{28}[j-1] &= F[j-1]\oplus A[o+j].
\end{aligned}
$$

消去相同的文章字节即可递推：

$$
F[j]=F[j-1]\oplus C_{33}[j]\oplus C_{28}[j-1].
$$

```python
from ast import literal_eval

def xor(left, right):
    return bytes(a ^ b for a, b in zip(left, right))

with open("result.log", "r", encoding="utf-8") as stream:
    data = [literal_eval(line.strip()) for line in stream if line.strip()]

assert len(data) == 100
assert len({len(item) for item in data}) == 1

known = b"moectf{"
article_fragments = [xor(item[:len(known)], known) for item in data]

# 人工比较 article_fragments 后可见：offset[28] = offset[33] + 1。
earlier = data[33]
later = data[28]

flag = bytearray(len(earlier))
flag[0] = ord("m")
for j in range(1, len(flag)):
    flag[j] = flag[j - 1] ^ earlier[j] ^ later[j - 1]

print(bytes(flag))
```

恢复结果为：

```text
moectf{W0W_y0U_HaVe_mastered_tHe_x0r_0Peart0r!_0iYlJf!M3rux9G9Vf!JoxiMl}
```

## 方法总结

同一明文与多个相关密钥流重复异或时，已知前缀既能泄露密钥流片段，也能帮助判断不同密钥流之间的相对位移。找到相差一字节的两个文章窗口后，相同文章字节会在异或中抵消，整个 flag 就能从一个已知首字节递推恢复。
