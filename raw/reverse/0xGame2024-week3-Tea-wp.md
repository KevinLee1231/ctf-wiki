# Tea

## 题目简述

附件是去除符号的 PE。通过 `memcmp` 附近的数据流可以定位输入、密文和密钥；加密函数中出现 32 轮循环以及常量 `0x61C88647`，它是 `-0x9E3779B9` 在 32 位整数中的表示，因此算法可识别为 TEA。程序还会先把 40 字节密文整体逆序，并且只对首尾两个 8 字节块执行了有效的 TEA 加密。

## 解题过程

主函数中，输入缓冲区经过加密后与静态数组比较，密钥是四个 32 位整数：

```text
key = [2, 0, 2, 4]
```

TEA 每轮对两个 32 位半块交替更新，典型 delta 为：

```text
0x9E3779B9
```

反汇编中的减法常量 `0x61C88647` 与之等价，因为：

$$
-0x61C88647\equiv0x9E3779B9\pmod{2^{32}}
$$

程序在比较前还调用了逆序函数，所以解密时先反转完整的 40 字节数组。输入缓冲区按 `uint32_t *` 使用，每次块指针增加 8 个 `uint32_t`，即 32 字节；在 40 字节有效范围内，实际对应偏移 `0` 和 `32` 的两个 8 字节块。循环后续迭代已经越界，只是干扰分析，不属于 flag 的有效数据。

使用小端序恢复两个 `uint32_t`，严格按模 $2^{32}$ 运算即可：

```python
import struct

ciphertext = bytearray([
    0xC9, 0xB6, 0x5C, 0xCE, 0xF8, 0xEE, 0x8E, 0xA2,
    0x33, 0x36, 0x34, 0x63, 0x37, 0x32, 0x36, 0x64,
    0x38, 0x37, 0x65, 0x33, 0x62, 0x33, 0x63, 0x64,
    0x36, 0x39, 0x64, 0x34, 0x64, 0x30, 0x62, 0x38,
    0x2A, 0x7A, 0x7C, 0x3B, 0x85, 0x33, 0x6D, 0xD3,
])

key = [2, 0, 2, 4]
delta = 0x9E3779B9
mask = 0xFFFFFFFF

def tea_decrypt(block: bytes) -> bytes:
    left, right = struct.unpack("<2I", block)
    total = (delta * 32) & mask

    for _ in range(32):
        right = (
            right
            - (
                (((left << 4) & mask) + key[2])
                ^ ((left >> 5) + key[3])
                ^ ((left + total) & mask)
            )
        ) & mask
        left = (
            left
            - (
                (((right << 4) & mask) + key[0])
                ^ ((right >> 5) + key[1])
                ^ ((right + total) & mask)
            )
        ) & mask
        total = (total - delta) & mask

    return struct.pack("<2I", left, right)

ciphertext.reverse()
ciphertext[0:8] = tea_decrypt(ciphertext[0:8])
ciphertext[32:40] = tea_decrypt(ciphertext[32:40])

print(ciphertext.decode())
```

输出为：

```text
0xGame{e8b0d4d96dc3b3e78d627c463c9953a1}
```

## 方法总结

TEA 的识别特征包括两个 32 位半块、四字密钥、32 轮 Feistel 式更新和黄金分割常量。还原时不能只套标准算法，还要复现题目额外的数据布局：整段逆序、小端 `uint32_t` 解释，以及相隔 32 字节的两个有效块。
