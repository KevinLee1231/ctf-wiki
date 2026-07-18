# week4茶谈室

## 题目简述

Android 应用把 40 字节输入按每 8 字节一组拆成两个 32 位大端整数，再使用固定 128 位密钥执行 32 轮 TEA 加密，与 10 个常量字比较。识别 TEA 后对 5 个分组执行逆运算即可恢复 flag。

## 解题过程

用 JEB 查看校验函数，可以看到典型常量 `0x9e3779b9`、32 轮循环、两个 32 位状态字以及四个密钥字。这些特征共同指向 TEA，而不是普通自定义异或。

密钥和密文常量为：

```python
key = [0x58C7BDF0, 0x413A4854, 0xBF51A817, 0x1F43A5E9]
cipher_words = [
    0x4FA7B4BE, 0xDE93990C,
    286770520, 0x97EBB430,
    0x5919D058, 0x7720C063,
    0xA0B696E3, 0x190A01D2,
    0x7D130AB9, 898703653,
]
```

按 TEA 的相反顺序先恢复 `v1`、再恢复 `v0`，每轮都截断到 32 位。程序按大端序组合字符，因此解密后也用大端序转回字节：

```python
mask = 0xffffffff
delta = 0x9e3779b9

def decrypt(v0, v1):
    total = (delta * 32) & mask

    for _ in range(32):
        v1 = (
            v1
            - (((v0 << 4) + key[2]) ^ (v0 + total) ^ ((v0 >> 5) + key[3]))
        ) & mask
        v0 = (
            v0
            - (((v1 << 4) + key[0]) ^ (v1 + total) ^ ((v1 >> 5) + key[1]))
        ) & mask
        total = (total - delta) & mask

    return v0, v1

plain = bytearray()
for index in range(0, len(cipher_words), 2):
    left, right = decrypt(cipher_words[index], cipher_words[index + 1])
    plain.extend(left.to_bytes(4, "big"))
    plain.extend(right.to_bytes(4, "big"))

print(plain.decode())
```

输出为：

```text
0xGame{ca5088de10c64b61ac1ef47a9d5f51da}
```

## 方法总结

本题的稳定指纹是 TEA 的 delta、32 轮和双 32 位状态更新。恢复时还必须匹配应用的分块方式与字节序；算法识别正确但把每个 32 位字按小端输出，仍会得到每 4 字节反转的错误文本。
