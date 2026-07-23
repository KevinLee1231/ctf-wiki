# LittLe_FSR

## 题目简述

题目把补齐到 50 字节的 `FLAG` 转成 400 位寄存器状态，使用 5 个未知抽头生成新位。状态先空转 2022 轮，再连续输出 2022 位：

```python
class LFSR:
    def __init__(self):
        self.data = list(
            map(int, bin(bytes_to_long(FLAG))[2:].rjust(400, "0"))
        )
        for _ in range(2022):
            self.cycle()

    def cycle(self):
        bit = self.data[0]
        new = 0
        for i in key:
            new ^= self.data[i]
        self.data = self.data[1:] + [new]
        return bit
```

题目的关键不在于枚举 $\binom{400}{5}$ 组抽头，而在于两件事：

1. LFSR 的输出满足 $GF(2)$ 上的线性递推，可以直接解出反馈系数；
2. 最小抽头不为 0 时，状态转移并非满秩，最前面的若干初始位不会影响任何后续输出。

## 解题过程

记连续输出为 $x_0,x_1,\ldots$，寄存器长度为 $n=400$，抽头集合为 $K$。按源码的左移方式，有

$$
x_{t+400}=\bigoplus_{k\in K}x_{t+k}.
$$

附件虽然从空转 2022 轮之后才开始记录，但线性递推与绝对时刻无关。设未知反馈向量为 $a=(a_0,\ldots,a_{399})$，使用 400 组连续输出即可建立方程

$$
\sum_{i=0}^{399}a_i x_{t+i}=x_{t+400},
\qquad 0\le t<400.
$$

在 $GF(2)$ 上求解得到

```text
[8, 23, 114, 211, 360]
```

最小抽头为 $a=8$。把递推式改写为

$$
x_{t+8}
=x_{t+400}
\bigoplus_{k\in K\setminus\{8\}}x_{t+k},
$$

便可以从已知的 $x_{2022},x_{2023},\ldots$ 逐步倒推到 $x_8$。但 $x_0$ 至 $x_7$ 从未出现在任何新位的计算中，经过移位后已经彻底丢失；仅凭密文无法恢复这 8 位。源码同时给出了 `FLAG[:7] == b"moectf{"`，因此可用已知格式补回首字节 `m`。

下面的 Sage 脚本同时完成抽头求解、递推校验和状态回退：

```python
from sage.all import GF, Matrix, vector

N = 400
WARMUP = 2022
stream = [
    int(bit)
    for bit in open("附件.txt", "r", encoding="utf-8").read().strip()
]

# stream[t + N] = sum(coeff[i] * stream[t + i]) over GF(2)
A = Matrix(GF(2), [stream[t:t + N] for t in range(N)])
b = vector(GF(2), stream[N:2 * N])
coeff = A.solve_right(b)
taps = [i for i, value in enumerate(coeff) if value]
print(taps)

# 使用未参与建方程的输出继续验证，避免窗口方向写反。
for t in range(N, len(stream) - N):
    predicted = sum(
        int(coeff[i]) * stream[t + i] for i in range(N)
    ) & 1
    assert predicted == stream[t + N]

smallest = min(taps)

# 附件的 stream[i] 对应原始序列 x[WARMUP + i]。
sequence = {
    WARMUP + i: bit
    for i, bit in enumerate(stream)
}

# 首次能向前求出的目标是 x[WARMUP - 1]。
for t in range(WARMUP - smallest - 1, -1, -1):
    value = sequence[t + N]
    for tap in taps:
        if tap != smallest:
            value ^= sequence[t + tap]
    sequence[t + smallest] = value

known_bits = "".join(
    str(sequence[i]) for i in range(smallest, N)
)
known_tail = int(known_bits, 2).to_bytes(len(known_bits) // 8, "big")

# 8 个不可观测位正好是首字节；50 字节状态末尾还有随机填充。
recovered = b"m" + known_tail
flag = recovered.split(b"}", 1)[0] + b"}"
print(flag)
```

输出为：

```text
moectf{Bru!e_@r_so1^e_A_sy3teM_0f_eq&aT10n3?}
```

## 方法总结

遇到已知连续输出的 LFSR，应先把异或视为 $GF(2)$ 上的加法，通过线性方程恢复反馈向量，再用额外输出验证窗口方向。恢复反馈多项式并不等于能恢复完整初始状态：还要检查状态转移是否可逆。本题最小抽头为 8，使最初 8 位对后续输出不可观测；这部分只能依靠题目明确给出的格式约束补全。
