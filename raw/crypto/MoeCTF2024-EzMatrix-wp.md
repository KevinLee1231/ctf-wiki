# EzMatrix

## 题目简述

题目实现 128 级 LFSR。初始 128 位状态就是 16 字节 secret，但程序先空转 128 次，再输出 256 bit，并隐藏最后 5 bit。需要枚举这 5 bit，用 $256=2n$ 个连续输出恢复未知反馈系数，再把状态逆推 128 步。

## 解题过程

设输出序列为 $a_i$，反馈系数为 $c_0,\ldots,c_{127}\in\mathbb F_2$，则：

$$
a_{i+128}=\sum_{t=0}^{127}c_ta_{i+t}\pmod2.
$$

对 $i=0,\ldots,127$ 收集方程，构造：

$$
M_{i,t}=a_{i+t},\qquad b_i=a_{i+128},\qquad M\vec c=\vec b.
$$

原稿写成了 `t M = t`，且把 128 位状态误称为 256 位寄存器；这里按源码修正。下面用整数 bitset 完成 $\mathbb F_2$ 高斯消元，不依赖 Sage：

```python
PREFIX = (
    "1111111001101101000011011010001111011011010111100010101100101011"
    "0011110011000011110001101011001100000011011101110000111001100111"
    "0111000101110011001111010100110001101101010111011000010101010110"
    "11101000110001111110100000011110010011010010100100000000110"
)
N = 128

def solve_gf2(sequence):
    rows = []
    for i in range(N):
        coefficients = sum(sequence[i + j] << j for j in range(N))
        rows.append(coefficients | (sequence[i + N] << N))

    where = [-1] * N
    rank = 0
    for column in range(N):
        pivot = next(
            (row for row in range(rank, N) if (rows[row] >> column) & 1),
            None,
        )
        if pivot is None:
            continue
        rows[rank], rows[pivot] = rows[pivot], rows[rank]
        for row in range(N):
            if row != rank and ((rows[row] >> column) & 1):
                rows[row] ^= rows[rank]
        where[column] = rank
        rank += 1

    for row in rows:
        if (row & ((1 << N) - 1)) == 0 and ((row >> N) & 1):
            return None
    if rank != N:
        return None
    return [(rows[where[column]] >> N) & 1 for column in range(N)]

for suffix in range(32):
    observed = list(map(int, PREFIX + f"{suffix:05b}"))
    coefficients = solve_gf2(observed)
    if coefficients is None or coefficients[0] != 1:
        continue

    # observed 是原始序列 a[128:384]。
    timeline = [None] * 384
    timeline[128:384] = observed

    # c[0] == 1 时，可由 a[i+128] 和后续 127 位反求 a[i]。
    for i in range(127, -1, -1):
        value = timeline[i + 128]
        for tap in range(1, 128):
            if coefficients[tap]:
                value ^= timeline[i + tap]
        timeline[i] = value

    secret = int("".join(map(str, timeline[:128])), 2).to_bytes(16, "big")
    if all(0x20 <= byte < 0x7F for byte in secret):
        print("missing bits:", f"{suffix:05b}")
        print("taps:", [i for i, bit in enumerate(coefficients) if bit])
        print(b"moectf{" + secret + b"}")
```

唯一可打印结果对应缺失位 `11010`，反馈 tap 为 `[0, 1, 19, 27]`，输出：

```text
moectf{e4sy_lin3ar_sys!}
```

## 方法总结

$n$ 级线性递推通常需要 $2n$ 个连续输出同时恢复反馈系数和状态。本题只少 5 bit，枚举 32 种补全后做线性方程求解即可；恢复系数后，利用常数项 $c_0=1$ 逐位反推，比显式计算 $128\times128$ 转移矩阵逆更直观。
