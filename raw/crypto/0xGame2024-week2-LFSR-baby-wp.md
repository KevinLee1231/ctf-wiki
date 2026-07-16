# LFSR-baby

## 题目简述

题目公开 128 位反馈掩码，并给出连续两段 128 位 LFSR 输出；未知量是初始状态 `seed`。由于掩码最高位为 1，状态转移可逆：把第一段输出视为 128 轮后的完整状态，逆推 128 次即可恢复初始 seed。

## 解题过程

正向更新为：

$$
o=s_0m_0\oplus s_1m_1\oplus\cdots\oplus s_{127}m_{127}
$$

$$
(s_0,s_1,\ldots,s_{127})\longrightarrow(s_1,s_2,\ldots,s_{127},o)
$$

已知新状态 $T=(t_0,\ldots,t_{127})$ 时，旧状态的 $s_1,\ldots,s_{127}$ 就是 $t_0,\ldots,t_{126}$。本题 $m_0=1$，所以：

$$
s_0=t_{127}\oplus\bigoplus_{i=1}^{127}s_i m_i
$$

固定输出为：

```text
Mask_seed = 245818399386224174743537177607796459213
O0        = 103763907686833223776774671653901476306
O1        = 136523407741230013545146835206624093442
```

注意 `O0` 是由 128 位字符串转成整数得到的，必须用 `:0128b` 补回开头的零；旧稿通过多逆推一轮碰巧补偿了这个问题，但不够稳健。完整脚本如下：

```python
from hashlib import md5

MASK_SEED = 245818399386224174743537177607796459213
O0 = 103763907686833223776774671653901476306
O1 = 136523407741230013545146835206624093442


def source_state(value):
    # 与题目 init_state 一致：缺少的位补在末尾
    bits = [int(bit) for bit in bin(value)[2:]]
    return bits + [0] * (128 - len(bits))


mask = source_state(MASK_SEED)
state = [int(bit) for bit in f"{O0:0128b}"]
assert mask[0] == 1

for _ in range(128):
    previous_first = state[-1]
    for index in range(127):
        previous_first ^= state[index] & mask[index + 1]
    state = [previous_first] + state[:-1]

seed = int("".join(str(bit) for bit in state), 2)
flag = "0xGame{" + md5(str(seed).encode()).hexdigest() + "}"

# 用两段公开输出验证恢复结果
check_state = state[:]
generated = []
for _ in range(256):
    output = 0
    for state_bit, mask_bit in zip(check_state, mask):
        output ^= state_bit & mask_bit
    check_state = check_state[1:] + [output]
    generated.append(str(output))

assert int("".join(generated[:128]), 2) == O0
assert int("".join(generated[128:]), 2) == O1

print(seed)
print(flag)
```

输出：

```text
326946364783555718644107160474855199220
0xGame{030ec00de18ceb4ddea5f6612d28bf39}
```

## 方法总结

已知反馈掩码且 $m_0=1$ 时，LFSR 的每一步都可以从新状态唯一还原旧状态。处理整数形式的定长比特串时还必须显式补齐前导零，否则状态会发生错位；用第二段输出正向复验可以及时发现此类问题。
