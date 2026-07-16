# EzLFSR

## 题目简述

题目使用 128 位线性反馈移位寄存器。未知 `mask` 正是 128 位 secret；每轮输出是当前 128 位状态与 mask 逐位与后再异或。题目给出初始状态及连续 256 个输出，因此前 128 个输出就能组成 $GF(2)$ 上的 128 元线性方程组并唯一求出 mask。

## 解题过程

第 $t$ 轮输出满足：

$$
y_t=\bigoplus_{i=0}^{127}(s_{t+i}\land k_i)
    =\sum_{i=0}^{127}s_{t+i}k_i\pmod2.
$$

其中状态序列前 128 位来自 `initState`，后续位就是已知 `outputState`。令矩阵第 $t$ 行为 $(s_t,s_{t+1},\ldots,s_{t+127})$，向量 $y$ 取前 128 个输出，则 $Mk=y$。

```python
# SageMath
from Crypto.Util.number import long_to_bytes

init_state = [...]       # task.py 中给出的 128 位列表
output_state = [int(x) for x in "..."]  # 题目输出的 256 位字符串

assert len(init_state) == 128
assert len(output_state) >= 128

states = init_state + output_state
M = Matrix(
    GF(2),
    [states[t:t + 128] for t in range(128)],
)
y = vector(GF(2), output_state[:128])
mask = M.solve_right(y)

bits = "".join(str(int(bit)) for bit in mask)
secret = long_to_bytes(int(bits, 2))
print(secret)
```

源码用 `bytes_to_long(secret)` 后 `zfill(128)` 生成 mask，因此求出的向量按索引 0 到 127 拼成二进制整数，再转回字节就是花括号内部的 secret；最终 flag 为 `b"0xGame{" + secret + b"}"`。

可使用剩余 128 个输出做验证：用恢复的 mask 继续计算，结果应与 `outputState[128:]` 完全一致。

## 方法总结

LFSR 的异或和按位与在 $GF(2)$ 上分别对应加法和乘法，所以足够多的连续输出会直接泄露线性反馈系数。构造矩阵时最容易出错的是窗口方向、行列顺序和 bit 到字节的顺序，应使用未参与求解的输出做独立校验。
