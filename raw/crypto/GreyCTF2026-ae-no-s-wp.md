# AE-no-S

## 题目简述

题目给出一个去除了 `SubBytes`（密钥扩展中的 `SubWord` 也变为恒等）的 AES 变种，以及公开 transcript：全零块的密文、全部 $128$ 个单比特基明文的密文和 PKCS#7 填充后 flag 的密文。其余轮函数仍包含 `AddRoundKey`、`ShiftRows` 与 `MixColumns`。

在 GF(2) 上，`ShiftRows` 是比特置换，`MixColumns` 是线性映射，`AddRoundKey` 是异或常量。移除唯一非线性的 SubBytes 后，固定密钥下的加密退化为一个仿射映射：

$$
E(P)=A(P)\oplus B.
$$

这里 $A$ 是 $128\times128$ 的二元线性映射，$B$ 是与密钥有关的常量偏移。恢复该仿射映射并解线性方程是决定性障碍，故归入 `crypto`。

## 解题过程

### 由零块和基向量恢复仿射变换

令 $e_i$ 为只有第 $i$ 个输入位为 $1$ 的 $16$ 字节块。全零输入直接给出偏移：

$$
B=E(0).
$$

因此每对公开基向量样本都给出了 $A$ 的一列：

$$
A(e_i)=E(e_i)\oplus E(0).
$$

要与 transcript 对齐，位必须按每字节 MSB-first 编号：首个基明文为 `80 00 ... 00`，最后一个为 `... 00 01`。将第 $i$ 个列向量的第 $r$ 位填入 Sage 行矩阵的第 $r,i$ 项；不能直接把列列表当成 Sage 的行列表。

```python
def block_to_bits(block):
    return [(byte >> shift) & 1 for byte in block for shift in range(7, -1, -1)]

columns = [None] * 128
for plaintext, ciphertext in basis_pairs:
    index = block_to_bits(plaintext).index(1)
    columns[index] = block_to_bits(xor_bytes(ciphertext, zero_ct))

rows = [
    [columns[input_bit][output_bit] for input_bit in range(128)]
    for output_bit in range(128)
]
matrix = Matrix(GF(2), rows)
assert matrix.rank() == 128
```

矩阵满秩是关键验证：它说明当前 bit order 与 transcript 解析正确，也说明每个密文块都能唯一反解。

### 解密 flag 密文并检查 padding

对于任意 flag 密文块 $C$，先减去仿射偏移，再解 $A(P)=C\oplus B$：

```python
def decrypt_block(ciphertext, zero_ct, matrix):
    target = vector(GF(2), block_to_bits(xor_bytes(ciphertext, zero_ct)))
    plaintext_bits = matrix.solve_right(target)
    out = bytearray(16)
    for i, bit in enumerate(plaintext_bits):
        if int(bit):
            out[i // 8] |= 1 << (7 - i % 8)
    return bytes(out)

plain_padded = b"".join(
    decrypt_block(flag_ct[i:i + 16], zero_ct, matrix)
    for i in range(0, len(flag_ct), 16)
)
pad_len = plain_padded[-1]
assert 1 <= pad_len <= 16 and plain_padded[-pad_len:] == bytes([pad_len]) * pad_len
flag = plain_padded[:-pad_len].decode()
```

padding 检查不是装饰：若基向量排序、转置方向或位序错了一处，解线性方程仍可能给出字节串，但 PKCS#7 结构会失败。官方 solver 的等价 Sage 实现输出：

```text
grey{iT5_4LL_l1N3R_aLGyBeR?_a1WaY5_HaZ_B1n...}
```

## 方法总结

- 核心技巧：删除代换层会把 AES 类轮函数从非线性密码变换降为 GF(2) 上的仿射变换；一个零样本与全体输入基样本足以完全恢复它。
- 识别信号：分组密码仅保留 XOR、固定置换、有限域线性列混合和轮常量，同时泄露 $E(0)$、$E(e_i)$ 时，应立即按仿射线性代数处理，而不是尝试常规 AES 攻击。
- 复用要点：明确输入/输出的 bit order，并在构造矩阵后检查秩。解得明文后用格式或 padding 独立验证，可快速定位转置和端序错误。
