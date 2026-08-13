# Cat And XOR Scam

## 题目简述

题目用一个 256 位 xorshift 状态生成异或密钥。状态更新依次为：

$$
x\mathrel{\oplus}=x\ll23,\qquad
x\mathrel{\oplus}=x\gg17,\qquad
x\mathrel{\oplus}=x\ll26
$$

每轮最后截断到 256 位。程序从固定种子 12345 初始化状态，连续推进 $2^{63}-1$ 轮，再把最终状态作为密钥与 flag 异或。逐轮模拟不可行，但 XOR、移位和截断在 $\operatorname{GF}(2)$ 上都是线性变换，因此可以对状态转移矩阵做快速幂。

公开仓库快照有一处实现不一致：种子扩展把每个状态字截成 128 位，后面却以 4 个 64 位字拼接和导出，`to_bytes(8, "big")` 会直接溢出。把种子扩展掩码修正为 64 位后，生成的密钥前缀与仓库 `flag.enc`、登记 flag 逐字节吻合；以下按这一预期的 4×64 位状态整理。

## 解题过程

先用 256 个基向量 $e_i$ 构造转移矩阵 $T$：对每个 $e_i$ 执行一次 xorshift，所得结果就是矩阵的第 $i$ 列。随后用平方乘计算 $T^{2^{63}-1}x_0$。

下面是去掉进度显示后的可运行核心脚本：

```python
from itertools import cycle
from pathlib import Path

WIDTH = 256
WORD_MASK = (1 << 64) - 1
STATE_MASK = (1 << WIDTH) - 1


def step(value):
    value ^= value << 23
    value ^= value >> 17
    value ^= value << 26
    return value & STATE_MASK


def apply(columns, value):
    result = 0
    while value:
        bit = value & -value
        result ^= columns[bit.bit_length() - 1]
        value ^= bit
    return result


def compose(left, right):
    # 返回 left ∘ right 的列表示。
    return [apply(left, column) for column in right]


def jump(value, distance):
    power = [step(1 << index) for index in range(WIDTH)]
    while distance:
        if distance & 1:
            value = apply(power, value)
        power = compose(power, power)
        distance >>= 1
    return value


def initial_state(seed):
    words = []
    value = seed
    for index in range(4):
        value ^= value >> 12
        value ^= value << 25
        value ^= value >> 27
        value = (value * 0x2545F4914F6CDD1D) & WORD_MASK
        words.append(value or 0x123456789ABCDEF0 + index)
    return sum(word << (64 * index) for index, word in enumerate(words))


state = jump(initial_state(12345), (1 << 63) - 1)
key = b"".join(
    ((state >> (64 * index)) & WORD_MASK).to_bytes(8, "big")
    for index in range(4)
)

ciphertext = bytes.fromhex(Path("flag.enc").read_text().strip())
plaintext = bytes(a ^ b for a, b in zip(ciphertext, cycle(key)))
print(plaintext.decode())
```

输出为：

```text
grey{5k1p_4h34d_1z_c4k5_fAv3}
```

## 方法总结

- 核心技巧：把 xorshift 表示成 $\operatorname{GF}(2)$ 上的线性矩阵，用平方乘跳过天文数量的状态更新。
- 识别信号：状态更新只含 XOR、固定位移和固定位宽截断，且轮数极大但初始状态已知。
- 复用要点：矩阵的行列方向、位序和状态字宽必须与实现一致；官方脚本也可能含注释或位宽错误，应先用小轮数和现有密文做一致性验证。
