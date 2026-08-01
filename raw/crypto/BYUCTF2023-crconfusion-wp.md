# BYUCTF 2023 - CRConfusion

## 题目简述

三个文本文件在相同位置各给出一段随机 32 位数据和一个 8 位 CRC。出题程序没有使用固定多项式，而是把 flag 每个字符的 ASCII 值当成对应位置的 8 位 CRC 多项式。

## 解题过程

对每一行位置，三个文件使用同一个未知多项式，但输入数据不同。把候选限制为可打印字符，对每个字符 `s` 计算三次 CRC；只有同时匹配三个校验值时才接受：

```python
import string

flag = ''
for a, b, c in zip(lines1, lines2, lines3):
    for ch in string.printable:
        poly = ord(ch)
        if (crc8(a.data, poly) == a.crc and
            crc8(b.data, poly) == b.crc and
            crc8(c.data, poly) == c.crc):
            flag += ch
            break
```

官方 `crc8` 实现先在数据后补 8 个零，每轮取最高位；若为 1，就把接下来的 8 位与候选多项式异或。三组独立样本用于排除单组数据可能产生的碰撞。最终恢复：

```text
byuctf{cyclic_redundancy_checks_are_used_to_detect_errors_in_data_transmission}
```

## 方法总结

这里不是逆向求一般 CRC 参数，而是利用参数空间只有约 95 个可打印字符。多份同位置样本把候选集合迅速压到唯一值；分析 oracle 时应主动寻找这种“参数复用、输入变化”的交叉约束。
