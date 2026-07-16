# Xor-Beginning

## 题目简述

程序把一段密文逐字节异或后输出。每个位置使用的异或值都不同，但它完全由下标决定，因此从反编译结果提取密文和循环表达式后即可还原明文。

## 解题过程

IDA 反编译得到核心循环：

```c
while (v4[v7]) {
    v4[v7] ^= 78 - (_BYTE)v7;
    ++v7;
}
```

第 $i$ 个密文字节使用 $78-i$ 作为异或值。异或满足：

$$
(A\oplus B)\oplus B=A
$$

所以加密和解密可以使用同一个运算。把程序中的密文字节抄出后逐项处理：

```python
data = [
    0x7E, 0x35, 0x0B, 0x2A, 0x27, 0x2C, 0x33, 0x1F, 0x76, 0x37,
    0x1B, 0x72, 0x31, 0x1E, 0x36, 0x0C, 0x4C, 0x44, 0x63, 0x72,
    0x57, 0x49, 0x08, 0x45, 0x42, 0x01, 0x5A, 0x04, 0x13, 0x4C,
]

plain = bytes(value ^ (78 - index) for index, value in enumerate(data))
print(plain.decode())
```

运行结果为：

```text
0xGame{X0r_1s_v3ry_Imp0rt4n7!}
```

## 方法总结

分析异或题时，需要同时提取密文、密钥生成方式和数据的实际字节顺序。本题的密钥流是下标函数 $78-i$，利用异或的自反性逐字节运算即可恢复 flag。
