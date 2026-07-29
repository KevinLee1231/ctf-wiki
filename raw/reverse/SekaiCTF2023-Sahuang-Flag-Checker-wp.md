# Sahuang Flag Checker

## 题目简述

这是一个要求 AVX-512 支持的 64 位 ELF。程序读取不超过 60 字节的输入，将其填充到 16 字节边界，然后对每个分组执行低半字节循环移位和 $16\times16$ 矩阵乘法，最后与内置的 64 字节密文比较。

SIMD 指令增加了反编译噪声，但核心变换发生在模 94 的整数环上。恢复矩阵、密文和半字节旋转方向后，可以直接求逆，无需暴力枚举 flag。

## 解题过程

`main` 使用 `%60s` 读取字符串，并反复附加 `X`，直到长度能被 16 整除。之后每 16 字节执行三步：

1. 字符减去 33，把可打印区间映射到 $0\ldots93$。
2. 保留高半字节，将低 4 位循环左移 3 位。
3. 与内置 $16\times16$ 矩阵 $M$ 相乘，对每项取模 94，再加回 33。

低半字节变换可写为：

```python
def rol4(value, count):
    value &= 0xF
    count %= 4
    return ((value << count) | (value >> (4 - count))) & 0xF

def transform_char(ch):
    value = ch - 33
    return (value & 0xF0) | rol4(value, 3)
```

反编译中用于浮点乘除和递归加减的辅助函数只是把整数余数计算拆得很复杂。结合目标范围可以把每组运算归纳为：

$$
c \equiv Mv \pmod{94},
$$

其中 $v$ 是 16 个旋转后的字符值，$c$ 是目标密文减 33 后的向量。内置密文共 64 字节：

```python
encrypted = bytes.fromhex(
    "2f4d3d6c64444c63506b57522a38733c"
    "46232f3d5c54494a3d2a625c29755934"
    "2d47254f22464563742247695b7d7b4a"
    "483e5b79436042616630707d283d2d74"
)
```

在环 $\mathbb{Z}/94\mathbb{Z}$ 上对 $M$ 做高斯消元，选取与 94 互素的可逆主元，可得到 $M^{-1}$。求逆后应先验证：

$$
MM^{-1}\equiv I\pmod{94}.
$$

逐组解密矩阵变换，再把低半字节循环右移 3 位，最后加回 33：

```python
def ror4(value, count):
    value &= 0xF
    count %= 4
    return ((value >> count) | (value << (4 - count))) & 0xF

plain = bytearray()

for offset in range(0, len(encrypted), 16):
    target = [
        value - 33
        for value in encrypted[offset:offset + 16]
    ]
    vector = [
        sum(matrix_inv[row][col] * target[col]
            for col in range(16)) % 94
        for row in range(16)
    ]
    plain.extend(
        ((value & 0xF0) | ror4(value & 0x0F, 3)) + 33
        for value in vector
    )

print(plain.decode())
```

输出为：

```text
SEKAI{1_I_i_|_H0oOo@p3eEe_Y0Uu\_/Didn't_BruT3F0rCe_GuYy5}XXXXXXX
```

末尾 7 个 `X` 是程序自动添加的分组填充，不属于 flag。因此答案是：

```text
SEKAI{1_I_i_|_H0oOo@p3eEe_Y0Uu\_/Didn't_BruT3F0rCe_GuYy5}
```

## 方法总结

SIMD 题先识别数据宽度和重复结构，通常比逐条翻译 intrinsic 更有效。本题的 AVX-512 只是并行实现：字符归一化、半字节置换、模 94 矩阵乘法。只要从比较常量恢复密文，在 $\mathbb{Z}/94\mathbb{Z}$ 中求矩阵逆，再施加相反的半字节旋转，就能确定全部输入；`X` 填充应在得到明文后按程序逻辑剥离。
