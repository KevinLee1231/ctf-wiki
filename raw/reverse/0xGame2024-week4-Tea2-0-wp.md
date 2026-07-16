# Tea2.0

## 题目简述

题目是一个 32 位 Windows 程序。`main` 读取 44 字节输入，将其按 6 个 64 位分组执行 64 轮 XTEA 加密，再与全局数组比较。容易忽略的是，PE 的 TLS 回调会在 `main` 之前修改密钥和目标数组，因此直接用反编译器中看到的静态数组做 XTEA 解密只会得到乱码。

## 解题过程

在 `main` 中可以确认：

- 输入长度固定为 `0x2c`，即 44 字节；
- `sub_41116D` 是 64 轮 XTEA 加密；
- XTEA 密钥为 `0x4512, 0x9832, 0x5647, 0x6314`；
- 程序把输入分成 6 组处理，最后一组的多余空间由缓冲区中的零补齐。

动态调试时，进入 `main` 后观察全局数组 `unk_41A020`，会发现它已经不等于文件中的硬编码值。继续检查 PE 的 TLS Directory 及该数组的交叉引用，可还原两个先于入口点执行的回调：

1. `TlsCallback_0_0` 将 TEA 密钥的四个 32 位字分别与 `0xABCD` 异或；
2. `TlsCallback_1_0` 使用修改后的密钥，对硬编码目标的 6 个分组分别执行 32 轮 TEA 加密。

因此正确的逆向顺序是：先对静态数组模拟 TLS 中的 TEA 加密，得到 `main` 实际比较的目标；再对该目标执行 64 轮 XTEA 解密。下面的脚本显式保留所有 32 位无符号整数溢出，并按 x86 小端序拼接结果：

```python
import struct

MASK = 0xFFFFFFFF
DELTA = 0x9E3779B9

def tea_encrypt(v0, v1, key):
    total = 0
    for _ in range(32):
        total = (total + DELTA) & MASK
        left = (((v1 << 4) & MASK) + key[0]) & MASK
        right = ((v1 >> 5) + key[1]) & MASK
        v0 = (v0 + (left ^ right ^ ((v1 + total) & MASK))) & MASK

        left = (((v0 << 4) & MASK) + key[2]) & MASK
        right = ((v0 >> 5) + key[3]) & MASK
        v1 = (v1 + (left ^ right ^ ((v0 + total) & MASK))) & MASK
    return v0, v1

def xtea_decrypt(v0, v1, key):
    total = (DELTA * 64) & MASK
    for _ in range(64):
        mix = ((((v0 << 4) & MASK) ^ (v0 >> 5)) + v0) & MASK
        v1 = (
            v1 - (mix ^ ((total + key[(total >> 11) & 3]) & MASK))
        ) & MASK
        total = (total - DELTA) & MASK
        mix = ((((v1 << 4) & MASK) ^ (v1 >> 5)) + v1) & MASK
        v0 = (
            v0 - (mix ^ ((total + key[total & 3]) & MASK))
        ) & MASK
    return v0, v1

data = [
    0x018DC360, 0xD5835457, 0x8BEE2DCB, 0x92BB2DEE,
    0xFDF4AD54, 0x043F8C2D, 0x61A232A9, 0x0F15F4D1,
    0x16EA4979, 0x7C2BF6DA, 0xDCD5FA32, 0x76450819,
]
tea_key = [value ^ 0xABCD for value in (0x1245, 0x3298, 0x4756, 0x1463)]
xtea_key = [0x4512, 0x9832, 0x5647, 0x6314]

for i in range(0, len(data), 2):
    data[i], data[i + 1] = tea_encrypt(data[i], data[i + 1], tea_key)

for i in range(0, len(data), 2):
    data[i], data[i + 1] = xtea_decrypt(data[i], data[i + 1], xtea_key)

plain = struct.pack("<12I", *data)
print(plain.rstrip(b"\x00").decode())
```

输出为：

```text
0xGame{a7961e4b-c809-f340-412e-91abd2c9b535}
```

## 方法总结

本题的关键机制是 TLS 回调早于 `main` 执行。分析带 TLS Directory 的 PE 时，静态全局变量不一定就是主逻辑实际使用的值，应在入口点处动态复核并检查所有写交叉引用。还原时必须同时保持 TEA/XTEA 的轮数、运算顺序、32 位溢出和小端序，否则即使算法名称判断正确也无法得到明文。
