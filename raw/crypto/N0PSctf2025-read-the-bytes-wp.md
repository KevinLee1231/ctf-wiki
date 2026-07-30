# N0PSctf2025 Read the bytes Writeup

## 题目简述

题目虽然标为 Reverse Engineering，但附件只是一个很短的 Python 程序：`flag` 是 `bytes` 对象，程序逐项遍历并打印每个元素。Python 3 遍历 `bytes` 时得到的是 $0$ 到 $255$ 的整数，因此输出实际上是 flag 各字节的十进制表示。决定性障碍是表示层解码，归入 Crypto。

## 解题过程

关键代码为：

```python
for char in flag:
    print(char)
```

题目同时给出了所有输出整数。对每个数调用 `chr()`，或先构造 `bytes`，即可恢复原文：

```python
values = [
    66, 52, 66, 89, 123, 52, 95, 67, 104, 52, 114, 97,
    67, 55, 51, 114, 95, 49, 115, 95, 74, 117, 53, 116,
    95, 52, 95, 110, 85, 109, 56, 51, 114, 33, 125,
]

print(bytes(values).decode())
```

输出为：

```text
B4BY{4_Ch4raC73r_1s_Ju5t_4_nUm83r!}
```

## 方法总结

- 核心技巧：把十进制字节序列还原为 ASCII/UTF-8 文本。
- 识别信号：Python 3 对 `bytes` 做迭代时输出整数，而不是长度为 1 的字符串。
- 复用要点：数值均在 $0$ 到 $255$ 时可直接用 `bytes(values)`；若超出该范围，应先确认它们是否是 Unicode 码点或其它编码。
