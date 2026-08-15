# CryptoCrack

## 题目简述

题目只提供一个 AMD64 ELF。逆向后可以找到两组各 35 字节的常量 `buff` 与 `key`，校验逻辑按下标奇偶采用两种变换：奇数位置是普通异或，偶数位置则在 AES 使用的有限域 $GF(2^8)$ 中进行乘法。

虽然计算中出现有限域运算，但决定性步骤是从二进制恢复常量和逐字节校验流程，因此归入 Reverse。得到变换关系后，每个字符都可独立求逆，不需要爆破完整 flag。

## 解题过程

有限域乘法使用不可约多项式 $x^8+x^4+x^3+x+1$；实现中左移溢出后异或 `0x1b`，与 AES 字节域一致。设第 $i$ 个密文字节为 $b_i$、常量为 $k_i$、明文字节为 $p_i$，逆向得到：

- 偶数下标：$p_i=b_i^{-1}\cdot k_i$；
- 奇数下标：$p_i=b_i\oplus k_i$。

其中乘法与求逆均在 $GF(2^8)$ 中进行。完整恢复脚本如下：

```python
buff = [
    0xA3, 0x97, 0xA2, 0x55, 0x53, 0xBE, 0xF1, 0xFC, 0xF9,
    0x79, 0x6B, 0x52, 0x14, 0x13, 0xE9, 0xE2, 0x2D, 0x51,
    0x8E, 0x1F, 0x56, 0x08, 0x57, 0x27, 0xA7, 0x05, 0xD4,
    0xD0, 0x52, 0x82, 0x77, 0x75, 0x1B, 0x99, 0x4A,
]

key = [
    0xD2, 0xFF, 0x8E, 0x39, 0x70, 0xD3, 0xEA, 0x88, 0x06,
    0x0A, 0x68, 0x15, 0xAD, 0x4C, 0x9E, 0xAD, 0x1D, 0x0E,
    0x85, 0x2B, 0x35, 0x38, 0x8C, 0x6E, 0x7A, 0x40, 0x19,
    0x8F, 0x1B, 0xB3, 0x8E, 0x34, 0x07, 0xFD, 0x4D,
]

def gf_mul(a, b):
    product = 0
    for _ in range(8):
        if b & 1:
            product ^= a
        carry = a & 0x80
        a = (a << 1) & 0xFF
        if carry:
            a ^= 0x1B
        b >>= 1
    return product

def gf_inverse(value):
    if value == 0:
        return 0
    for candidate in range(1, 256):
        if gf_mul(value, candidate) == 1:
            return candidate
    raise ValueError("no inverse")

plain = []
for i, (b, k) in enumerate(zip(buff, key)):
    if i % 2 == 0:
        plain.append(gf_mul(gf_inverse(b), k))
    else:
        plain.append(b ^ k)

print(bytes(plain).decode())
```

运行结果为：

```text
shellmates{Gg_yOu_n4N0mITES_w1ZARd}
```

也可以对偶数位置枚举可打印字符，测试 `gf_mul(candidate, buff[i]) == key[i]`。但直接计算乘法逆元更清楚，也说明了变换为什么可逆。

## 方法总结

分析这类校验程序时，应先把循环按下标、分支和运算域还原成数学关系，再选择逆运算。异或是自身的逆；有限域中的非零元素都有唯一乘法逆元，所以偶数位置同样可以逐字节直接恢复。

需要特别区分普通整数乘法与有限域乘法。这里的中间结果始终限制在 8 位，并以 `0x1b` 约减；若误用模 256 整数乘法，求出的逆元和明文都会错误。
