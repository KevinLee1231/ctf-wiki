# d1ffer3nce

## 题目简述

题目附件是一个经过 UPX 压缩的 Go 可执行程序。脱壳后可以确认其核心是修改过轮数和 Delta 常量的 XXTEA：密钥由 Go 字符串常量拼接得到，程序加密输入后与内置的 32 字节密文比较。

决定性障碍是从 Go 二进制中恢复修改版 XXTEA 的参数并执行逆运算，因此归类为 Reverse。

## 解题过程

### 脱壳并识别密钥

附件带有 UPX 特征，可以先执行常规 UPX 脱壳，再分析展开后的 Go 程序。Go 编译产物中字符串常量常以相邻数据保存，反编译结果也可能表现为多段字符串拼接；结合显式的长度参数 16，可以恢复实际使用的密钥：

```text
0123456789abcdef
```

### 确认 XXTEA 的修改点

加密循环保留了 XXTEA 的数据相关混合形式，但有两处参数不再使用标准值：

- Delta 被改为 `0x4D696E69`，按大端字符观察即 `Mini`；
- 轮数为 `6 + 2025 // n`，其中 $n$ 是 32 位字的个数。

目标密文是：

```text
729daebea2e3845b310f01f1b3e703c24c810a9ca0ed2c4d9252a214882d7721
```

密文共 32 字节，按小端序拆成 8 个 32 位字，因此实际轮数为

$$
6+\left\lfloor\frac{2025}{8}\right\rfloor=259.
$$

解密仍按 XXTEA 的逆序更新方式进行：初始累加和为

$$
sum=259\times 0x4D696E69 \pmod {2^{32}},
$$

每轮从最后一个字向第一个字更新，并在轮末减去修改后的 Delta。所有加减法和移位结果都必须截断为 32 位无符号整数。

### 独立解密脚本

下面的脚本只使用 Python 标准库，并显式实现题目的修改版 XXTEA：

```python
import struct

MASK = 0xFFFFFFFF
DELTA = 0x4D696E69


def mx(total, y, z, p, e, key):
    left = ((z >> 5) ^ ((y << 2) & MASK))
    right = ((y >> 3) ^ ((z << 4) & MASK))
    return ((left + right) ^
            ((total ^ y) + (key[(p & 3) ^ e] ^ z))) & MASK


def decrypt_xxtea(data, key_bytes):
    words = list(struct.unpack(f"<{len(data) // 4}I", data))
    key = list(struct.unpack("<4I", key_bytes))
    n = len(words)
    rounds = 6 + 2025 // n
    total = (rounds * DELTA) & MASK
    y = words[0]

    while total:
        e = (total >> 2) & 3
        for p in range(n - 1, 0, -1):
            z = words[p - 1]
            words[p] = (words[p] - mx(total, y, z, p, e, key)) & MASK
            y = words[p]

        z = words[n - 1]
        words[0] = (words[0] - mx(total, y, z, 0, e, key)) & MASK
        y = words[0]
        total = (total - DELTA) & MASK

    return struct.pack(f"<{n}I", *words)


ciphertext = bytes.fromhex(
    "729daebea2e3845b310f01f1b3e703c2"
    "4c810a9ca0ed2c4d9252a214882d7721"
)
plaintext = decrypt_xxtea(ciphertext, b"0123456789abcdef")
print(repr(plaintext))
print(plaintext[:-plaintext[-1]].decode())
```

输出为：

```text
b'miniLCTF{W3lc0m3~MiN1Lc7F_2O25}\x01'
miniLCTF{W3lc0m3~MiN1Lc7F_2O25}
```

末尾的 `0x01` 是一个字节的 PKCS#7 填充，去除后即为提交内容。

## 方法总结

分析魔改算法时，应先把标准算法的结构与题目实现逐项对齐，而不是直接套用默认库函数。本题必须同时恢复小端字序、16 字节密钥、Delta 和轮数公式；漏掉任一项都会得到完全不同的明文。解密结果末尾还能看到合法的 `0x01` 填充，为参数恢复提供了额外验证。

最终 flag 为：

```text
miniLCTF{W3lc0m3~MiN1Lc7F_2O25}
```
