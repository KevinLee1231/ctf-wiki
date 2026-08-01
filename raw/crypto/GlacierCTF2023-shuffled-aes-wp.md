# GlacierCTF2023 - Shuffled AES

## 题目简述

题目将 AES 每轮组件错误地分组：先连续执行 10 轮 `SubBytes + AddRoundKey`，再连续执行 10 轮 `ShiftRows + MixColumns + AddRoundKey`，并把该分组密码用于 12 字节 nonce 加 4 字节大端计数器的 CTR 模式。服务会返回加密 flag，并提供任意明文加密 oracle。

## 解题过程

后半段 10 轮只有线性/仿射运算。对已知明文查询，用明文异或密文先取出 CTR 密钥流，再连续应用 10 次 `InvMixColumns`、`InvShiftRows`，即可移除后半段的线性扩散。

前半段没有 `ShiftRows` 或 `MixColumns`，所以 16 个字节位置彼此独立。对位置 $j$ 建表

$$
L_j[x]=F_j(x),
$$

其中 $x$ 是 `nonce || counter` 的第 $j$ 个字节，$F_j$ 是十轮逐字节 `SubBytes + AddRoundKey` 的合成。每次 oracle 返回随机 nonce，能同时采样多个位置；反复查询，直到目标 flag 的每个输入字节都已出现在相应表中。

```python
tables = [dict() for _ in range(16)]

while True:
    q_nonce, stream = get_known_plaintext_keystream()
    for block_idx, block in enumerate(blocks(stream)):
        inp = q_nonce + block_idx.to_bytes(4, "big")
        mid = bytearray(block)
        for _ in range(10):
            inv_mix_columns(mid)
            inv_shift_rows(mid)
        for j in range(16):
            tables[j][inp[j]] = mid[j]

    if all(flag_inputs_are_covered(tables)):
        break
```

解密 flag 时，从各表取出中间态，再正向执行 10 次 `ShiftRows`、`MixColumns`，得到 flag nonce 对应的 CTR 密钥流并与密文异或。最终得到：

```text
gctf{c0nfU510n_AnD_D1fFU510N_Mu57_n07_83_53pARA73d}
```

## 方法总结

分组密码的安全性依赖非线性混淆与线性扩散在轮间交错。把它们分开会使后半段可逆、前半段退化成 16 个独立的 8 位置换；选择明文 oracle 随即可以用查表恢复整个 CTR 密钥流。
