# oooollvm

## 题目简述

程序经过 OLLVM 控制流平坦化和指令替换，但有效代码块中的校验本身很短：对输入的第 $i$ 个字符与 `table1[i] + i` 做按位选择运算，再与 `table2[i]` 比较。该表达式本质上就是逐字节异或，因此无需完整恢复所有分发器状态。

## 解题过程

可以使用 deflat 辅助去平坦化，也可以在调试器中只跟踪会改变计数器和比较结果的真实块。最终得到两张 34 字节表：

```python
table2 = [
    0x77, 0x25, 0x71, 0x3F, 0xF1, 0x46, 0xAB, 0x4F,
    0x5F, 0x7E, 0x87, 0x89, 0x3E, 0x89, 0x24, 0x17,
    0x5C, 0x19, 0xA1, 0x36, 0xD2, 0x3C, 0x72, 0x51,
    0x21, 0x9C, 0xB7, 0xA5, 0xD0, 0x9A, 0x1A, 0x77,
    0x06, 0x3A,
]

table1 = [
    0x1F, 0x41, 0x0E, 0x4F, 0x90, 0x38, 0x95, 0x1C,
    0x2B, 0x1F, 0xC0, 0xCB, 0x03, 0xAF, 0x6D, 0x45,
    0x5C, 0x63, 0xBF, 0x67, 0x83, 0x4F, 0x16, 0x1C,
    0x3C, 0xAF, 0xAF, 0x75, 0x9D, 0xBA, 0x2C, 0x1C,
    0x43, 0x26,
]
```

每一位的校验可写成：

```text
table2[i] == ((~q & k) | (~k & q)),  k = table1[i] + i
```

对单个位观察可知：当 `k` 为 1 时结果取 `~q`，当 `k` 为 0 时结果取 `q`，这正是 `q XOR k`。由于反编译表达式可能带有更宽的整数类型，求解时应显式限制到 8 位。枚举可打印字符即可避免符号扩展歧义：

```python
answer = []

for i, target in enumerate(table2):
    k = (table1[i] + i) & 0xff
    matches = []

    for q in range(0x20, 0x7f):
        value = (
            (((~q) & 0xff) & k)
            | (((~k) & 0xff) & q)
        )
        if value == target:
            matches.append(q)

    assert len(matches) == 1
    answer.append(matches[0])

print(bytes(answer).decode())
```

输出为：

```text
hgame{0llVM_15_C0mpLEX^buT~5iMPLe}
```

原 PDF 只给出“有效块最终是简单 xor”的结论，没有抄出两张常量表。本文采用公开复盘中的常量，并独立运行求解器验证了每一位都有唯一可打印解；补充来源为 [HGAME2020 RE/PWN 复盘](https://blog.51cto.com/u_14601424/6286037)。

## 方法总结

- 核心技巧：把精力集中在真实比较块和数据依赖上，不必为了一个短校验器完整还原所有 OLLVM 分发器。
- 关键化简：`(~q & k) | (~k & q)` 等价于按位异或 `q ^ k`。
- 复用要点：Python 的 `~` 产生无限精度负数，模拟 C 字节运算时必须用 `& 0xff` 截断。
