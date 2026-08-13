# Greycademy2025 Artifact 15: Bitwise

## 题目简述

附件是一段 Python 位运算练习。程序先要求识别 `mystery` 的功能，再对 41 个 flag 字节分别执行 `mystery` 和循环左移，与硬编码数组逐项比较。

## 解题过程

逐位观察 `mystery(x, y)`：循环取出 `x` 和 `y` 的同一位，只有两位不同时才在结果中置位，因此它等价于一字节异或 `x ^ y`。第一问输入：

```text
xor
```

`rol(x, n)` 每次把最高位绕回最低位，是 8 位循环左移。由于每个 flag 字节只与固定的 key 字节、旋转次数和 check 值发生关系，各位置完全独立；每个位置枚举 0 到 255 即可：

```python
def mystery(x, y):
    return x ^ y

def rol8(x, n):
    n %= 8
    return ((x << n) | (x >> (8 - n))) & 0xff

key = b"merry christmas"
check = [
    40, 139, 226, 176, 128, 74, 139, 141, 176, 177,
    68, 142, 35, 66, 65, 100, 22, 2, 99, 142, 69,
    17, 115, 24, 8, 194, 178, 3, 160, 60, 30, 0,
    0, 75, 5, 40, 16, 56, 46, 99, 7,
]
offsets = [
    10, 15, 53, 60, 54, 35, 55, 63, 36, 27, 6,
    23, 37, 5, 21, 57, 24, 16, 13, 63, 40, 32, 52,
    2, 51, 21, 20, 48, 67, 1, 57, 30, 50, 6, 46,
    21, 56, 1, 57, 45, 31,
]

flag = bytearray()
for i, expected in enumerate(check):
    matches = [
        c for c in range(256)
        if rol8(mystery(c, key[i % len(key)]), offsets[i]) == expected
    ]
    assert len(matches) == 1
    flag.append(matches[0])

print(flag.decode())
```

输出并由原程序验证：

```text
grey{itsy_bitsy_spider_the_number_master}
```

## 方法总结

复杂位运算应先验证其逐位真值，而不是被代码形状吓住；`mystery` 最终只是 XOR，旋转次数也只需对 8 取模。更重要的是识别约束独立性：41 个字节没有相互反馈，搜索空间是 41 次 256，而不是 $256^{41}$。枚举后断言每位只有一个候选，可同时验证分析和常量抄录是否正确。
