# upx magic 0

## 题目简述

题目原计划考查 UPX，但比赛时误放了未加壳附件，因此可直接静态分析。程序要求输入 32 个字符，对每个字符独立执行 8 轮、生成多项式为 `0x1021` 的 CRC16 变换，再与 32 个常量逐项比较。

## 解题过程

校验函数先把一个输入字节左移 8 位，然后重复 8 次：

```c
crc = input << 8;
for (int bit = 0; bit < 8; bit++) {
    if (crc & 0x8000)
        crc = (crc << 1) ^ 0x1021;
    else
        crc <<= 1;
}
```

每个明文字节相互独立，所以无需逆多项式或爆破整串 flag。枚举可见 ASCII，建立 `CRC -> 字符` 映射，再逐项查表即可：

```python
TARGET = [
    0x8d68, 0x9d49, 0x2a12, 0xab1a, 0xcbdc, 0xb92b, 0x2e32, 0x9f59,
    0xddcd, 0x9d49, 0xa90a, 0x0e70, 0xf5cf, 0x0a50, 0x5af5, 0xff9f,
    0x9f59, 0xbd0b, 0x58e5, 0x3823, 0xbf1b, 0x78a7, 0xab1a, 0x48c4,
    0xa90a, 0x2c22, 0x9f59, 0x5cc5, 0x5ed5, 0x78a7, 0x2672, 0x5695,
]

def crc16_byte(value: int) -> int:
    crc = value << 8
    for _ in range(8):
        if crc & 0x8000:
            crc = (crc << 1) ^ 0x1021
        else:
            crc <<= 1
    return crc & 0xffff

lookup = {crc16_byte(value): chr(value) for value in range(0x20, 0x7f)}
plain = "".join(lookup[item] for item in TARGET)
print(f"hgame{{{plain}}}")
```

输出为：

```text
hgame{noW_YOu~koNw-UPx~mAG|C_@Nd~crC16}
```

## 方法总结

CRC 的目标是检错，不是保密。此题还把每个字节单独映射成 16 位结果，完全没有跨字符扩散，因此搜索复杂度只有“字符集大小乘字符串长度”。识别此类循环时应关注最高位测试、左移和常见多项式 `0x1021`；即使静态链接去掉了符号，也能从运算结构判断算法。
