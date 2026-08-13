# AES

## 题目简述

服务端允许提交一段明文并返回自定义 AES 的 ECB 密文，同时给出随机 16 字节密码的密文。只有还原密码才能取得 flag。实现保留了 `SubBytes`、`ShiftRows` 和轮密钥加，但把真正提供扩散性的 `MixColumns` 错写成了列内循环移位，因此每个输出字节始终只依赖一个输入字节。

## 解题过程

题目中的伪 `MixColumns` 如下：

```python
def mix_single_column(a):
    a[0], a[1], a[2], a[3] = a[1], a[2], a[3], a[0]
```

它只是置换位置，没有进行有限域乘法与异或。`SubBytes` 是逐字节置换，`AddRoundKey` 也是逐字节异或；结合 `ShiftRows` 后，整个加密仍可看作 16 条彼此独立的单字节映射，只是输入与输出位置经过固定置换。

先用与题目相同的状态布局模拟 9 轮 `ShiftRows + fake MixColumns`，再补上末轮 `ShiftRows`，得到输入位置到输出位置的映射。随后一次提交 256 个明文块，其中第 $i$ 块为 `bytes([i]) * 16`：

```python
mapping = [[i * 4 + j for j in range(4)] for i in range(4)]
for _ in range(1, 10):
    shift_rows(mapping)
    mix_columns(mapping)
shift_rows(mapping)
mapping = [mapping[i][j] for i in range(4) for j in range(4)]

payload = b"".join(bytes([i]) * 16 for i in range(256))
```

服务端会按 PKCS#7 规则额外加一个填充块，因此解析响应时丢弃最后 16 字节。对于每个输出位置，把 256 个密文字节反查到对应的输入值，建立查表关系：

```python
blocks = [ciphertext[i:i + 16]
          for i in range(0, len(ciphertext) - 16, 16)]
dec_map = [[-1] * 256 for _ in range(16)]

for value in range(256):
    for out_pos in range(16):
        in_pos = mapping[out_pos]
        dec_map[in_pos][blocks[value][out_pos]] = value

password = bytearray(16)
for out_pos in range(16):
    in_pos = mapping[out_pos]
    password[in_pos] = dec_map[in_pos][password_ct[out_pos]]
```

提交恢复出的 16 字节密码，得到：

```text
grey{mix_column_is_important_in_AES_ExB3Hf9q9I3m}
```

## 方法总结

AES 的安全性不仅来自 S-box，还依赖 `MixColumns` 在字节之间建立扩散。若线性混合被退化为纯位置置换，选择明文即可为每个位置独立建立完整的 256 项逆向表。分析自定义分组密码时，应先判断各轮是否真正让一个输入字节影响多个输出字节，再决定是否需要攻击完整轮结构。
