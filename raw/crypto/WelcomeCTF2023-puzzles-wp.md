# puzzles

## 题目简述

Flag 被分为五个 8 字节块 $P_0,\ldots,P_4$，使用同一个 8 字节随机密钥 $K$ 按环形相邻关系加密：

$$
C_i=P_i\oplus P_{i+1}\oplus K,
$$

其中下标按模 5 计算。虽然 $K$ 随机，但五条方程并不独立安全；再结合已知 Flag 前缀 $P_0=\texttt{greyhats}$，可以逐块消元。

## 解题过程

附件输出先做 Base64 解码，再按 8 字节切成五个密文块。由相邻方程异或可消去密钥，例如：

$$
C_0\oplus C_1=P_0\oplus P_2,
$$

$$
C_3\oplus C_4=P_3\oplus P_0.
$$

因此已知 $P_0$ 后可以按链式关系恢复全部明文：

```python
import base64

def xor(a, b):
    return bytes(x ^ y for x, y in zip(a, b))

raw = base64.b64decode(
    "+hpPlyY9EZf7WVLdKgMI198cSt0gAyLmiE4Bgj19bqiwSRSkJixFtQ=="
)
C = [raw[i:i + 8] for i in range(0, len(raw), 8)]
P = [None] * 5
P[0] = b"greyhats"
P[2] = xor(xor(C[0], C[1]), P[0])
P[3] = xor(xor(C[3], C[4]), P[0])
P[4] = xor(xor(C[2], C[3]), P[2])
P[1] = xor(xor(C[1], C[2]), P[3])
print(b"".join(P).decode())
```

输出为：

```text
greyhats{0h_y0u_f1x3d_m3_up_s0_n1c3ly!!}
```

## 方法总结

- 核心技巧：将重复密钥的环形 XOR 方程两两相消，再用已知明文块固定解。
- 识别信号：同一随机密钥出现在多条线性 XOR 关系中，且 Flag 格式提供完整分块前缀。
- 复用要点：把加密过程写成线性方程组比盲猜更可靠；先检查变量和方程之间是否存在可消去密钥的组合。
