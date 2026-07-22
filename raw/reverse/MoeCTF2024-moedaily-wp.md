# moedaily

## 题目简述

附件把校验逻辑藏在 Excel 工作簿的 `s3cr3t` 工作表中。公式用 `BITLSHIFT`、`BITRSHIFT`、`BITXOR` 和 `BITAND(..., 4294967295)` 实现 32 位 TEA，并对每个 8 字节块连续加密两次。

## 解题过程

`Sheet1!D12` 先检查输入长度为 48，再把六组结果与常量比较。`Sheet1!H14:I19` 分别引用 `s3cr3t!E54:F54`、`L54:M54`、`S54:T54`、`E72:F72`、`L72:M72`、`S72:T72`，对应六个 64 位密文块：

```python
ciphertext = [
    (1397140385, 2386659843),
    (962571399, 3942687964),
    (3691974192, 863943258),
    (216887638, 3212824238),
    (3802077983, 1839161422),
    (1288683919, 3222915626),
]
```

公式中的四个 key word 是 `(114514, 1919810, 415144, 19883)`，TEA 的 delta 被改成 `114514`。每个公式区前 16 行更新一对 word，旁边另一组列再接着完成后 16 轮；下半区又以上半区密文为输入重复完整 32 轮，所以解密函数需要调用两次。

```python
from struct import pack

MASK = 0xFFFFFFFF
DELTA = 114514
KEY = (114514, 1919810, 415144, 19883)

ciphertext = [
    (1397140385, 2386659843),
    (962571399, 3942687964),
    (3691974192, 863943258),
    (216887638, 3212824238),
    (3802077983, 1839161422),
    (1288683919, 3222915626),
]

def tea_decrypt(v0, v1):
    total = (DELTA * 32) & MASK
    for _ in range(32):
        mix1 = (((v0 << 4) & MASK) + KEY[2]) \
             ^ ((v0 + total) & MASK) \
             ^ ((v0 >> 5) + KEY[3])
        v1 = (v1 - mix1) & MASK

        mix0 = (((v1 << 4) & MASK) + KEY[0]) \
             ^ ((v1 + total) & MASK) \
             ^ ((v1 >> 5) + KEY[1])
        v0 = (v0 - mix0) & MASK
        total = (total - DELTA) & MASK
    return v0, v1

plain = bytearray()
for block in ciphertext:
    block = tea_decrypt(*block)
    block = tea_decrypt(*block)
    plain += pack("<II", *block)

print(plain.decode())
```

小端拼接六个明文块，输出：

```text
moectf{3xC3l_1S_n0t_just_f0r_d41ly_w0rk_bu7_R3V}
```

## 方法总结

分析表格型逆向题不能只看最终比较常量，应沿跨工作表引用还原数据流。这里 `BITAND(..., 0xffffffff)` 明确了 32 位截断，列间引用揭示每次 16 轮的分段，而第二组公式以上一组输出为输入，证明 TEA 被完整执行了两遍。
