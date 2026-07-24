# UMDCTF 2019 - Silver

## 题目简述

题目只给出标题 `Silver` 和一段密文。这里的标题不是装饰：银的原子序数是 $47$，它直接给出了需要尝试的 ROT 编号。

## 解题过程

普通 ROT13 只在 26 个英文字母之间轮换，而 ROT47 会在 ASCII 可打印字符 `!` 到 `~` 之间旋转 47 位，因此还能同时处理数字和标点。对题目字符串逐字符执行 ROT47：

```python
def rot47(text: str) -> str:
    result = []
    for char in text:
        value = ord(char)
        if 33 <= value <= 126:
            value = 33 + (value - 33 + 47) % 94
        result.append(chr(value))
    return "".join(result)
```

解码结果为：

```text
UMDCTF-{Wh@t_c0mes_@fter_47?}
```

对该字符串计算 SHA-256，与仓库 `README.md` 中保存的摘要一致。

## 方法总结

这题的关键是把标题提供的元素信息转换为原子序数，再据此选择 ROT47。遇到包含大量数字和标点、又不符合 ROT13 特征的可打印 ASCII 密文时，应优先考虑 ROT47，而不是只在字母表范围内做凯撒位移。
