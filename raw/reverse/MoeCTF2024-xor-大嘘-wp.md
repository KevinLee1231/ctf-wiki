# xor（大嘘）

## 题目简述

程序表面上充满 XOR 和花指令，实际数据流是三层：两个 32 字节常量先逐字节异或，结果按四个 8 字节块做标准 TEA 解密，最后再与 16 字节字符串 `hello_moectf2024` 循环异或。

## 解题过程

TEA key 就是 `hello_moectf2024` 按小端序拆成四个 `uint32_t`。将花指令去掉后，可以直接复现完整逆向流程：

```python
from struct import pack, unpack

MASK = 0xFFFFFFFF
DELTA = 0x9E3779B9
key_bytes = b"hello_moectf2024"
key = unpack("<4I", key_bytes)

left = bytes([
    0x2B, 0xF2, 0x82, 0x41, 0x48, 0x74, 0x9D, 0xAA,
    0x7E, 0x4C, 0xDA, 0x04, 0x08, 0x2C, 0xA8, 0x52,
    0x97, 0x77, 0xB7, 0x3B, 0x16, 0x2D, 0xD4, 0xFC,
    0x60, 0xBE, 0xC4, 0xB6, 0x73, 0x19, 0x94, 0x87,
])
right = bytes([
    0x3C, 0x0D, 0x05, 0x1F, 0x30, 0x6E, 0x1E, 0x30,
    0x04, 0x3C, 0x12, 0x52, 0x59, 0x03, 0x6D, 0x52,
    0x04, 0x04, 0x0B, 0x33, 0x1F, 0x33, 0x17, 0x3B,
    0x17, 0x1A, 0x2B, 0x07, 0x55, 0x04, 0x5B, 0x5A,
])

def tea_decrypt(block):
    v0, v1 = unpack("<2I", block)
    total = (DELTA * 32) & MASK
    for _ in range(32):
        mix1 = (((v0 << 4) + key[2]) ^ (v0 + total)
                ^ ((v0 >> 5) + key[3]))
        v1 = (v1 - mix1) & MASK
        mix0 = (((v1 << 4) + key[0]) ^ (v1 + total)
                ^ ((v1 >> 5) + key[1]))
        v0 = (v0 - mix0) & MASK
        total = (total - DELTA) & MASK
    return pack("<2I", v0, v1)

mixed = bytes(a ^ b for a, b in zip(left, right))
decoded = b"".join(
    tea_decrypt(mixed[offset:offset + 8])
    for offset in range(0, len(mixed), 8)
)
plain = bytes(
    value ^ key_bytes[index % len(key_bytes)]
    for index, value in enumerate(decoded)
)
print(plain.decode())
```

输出：

```text
moectf{how_an_easy_junk_and_tea}
```

## 方法总结

花指令分析的重点是追踪真正参与最终比较的数据依赖。题目名中的 XOR 只覆盖最外两层，决定性结构仍是 32 轮、delta 为 `0x9e3779b9` 的 TEA；识别后按“外层 XOR → TEA 解密 → 最后一层 XOR”的逆序处理即可。
