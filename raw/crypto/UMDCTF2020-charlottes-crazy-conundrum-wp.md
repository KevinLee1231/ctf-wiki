# Charlotte's Crazy Conundrum

## 题目简述

题目把 flag 的每个字节拆成高、低两个半字节，再分别映射为 `a` 到 `p`。得到的字母序列还要按照题目提供的 Pigpen 字形表解释，因此需要依次逆转“图形码表”和“半字节编码”两层表示。

## 解题过程

先根据 Pigpen JSON 中每个图形与字母的对应关系，把图形密文还原为只包含 `a` 至 `p` 的字符串。对相邻两个字符分别减去 `ord("a")`，第一个作为高 4 位，第二个作为低 4 位：

```python
def decode_pair(left: str, right: str) -> int:
    high = ord(left) - ord("a")
    low = ord(right) - ord("a")
    return (high << 4) | low

encoded = (
    "ffeneeedfeegcnhldbfpgmdahgddfphadbghhaddgofpgddbh"
    "agiddhcdbgoghhn"
)
plain = bytes(
    decode_pair(encoded[i], encoded[i + 1])
    for i in range(0, len(encoded), 2)
)
print(plain.decode())
```

这里 `a` 表示 0，`p` 表示 15，因此每两个字符恰好重组一个原始字节。解码结果为：

```text
UMDCTF-{1_l0v3_p1gp3n_c1ph3r1ng}
```

## 方法总结

这类题要把“视觉符号”和“实际数据编码”分层处理：Pigpen 只负责把图形还原成字母，而 `a` 至 `p` 才是承载字节的十六进制式码表。先确认每层的输入输出域，可以避免把两套规则混在一起。
