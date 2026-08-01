# BYUCTF 2022 - Shouty 2

## 题目简述

题目提供 AIFF 与 M4A 两份相同内容的音频。录音反复读出汉语数字“零”和“一”，本质上是把二进制位逐位念了出来。

## 解题过程

将每个发音记为一位：`líng` 对应 `0`，`yī` 对应 `1`。仓库中的 `shouty.txt` 也保存了同一串位流，可用来校对人工听写结果。每 8 位按高位在前组成一个字节，再按 ASCII 解码：

```python
bits = bits.replace(" ", "").replace("\n", "")
plain = bytes(int(bits[i:i + 8], 2) for i in range(0, len(bits), 8))
print(plain.decode())
```

解码结果是一段抱怨手工计数的英文句子，末尾直接给出：

```text
byuctf{i_hate_counting_so_much_cl3XYZLn}
```

若输出完全不可读，应优先检查两点：是否把零和一映射反了，以及是否从第一位开始严格按 8 位分组。

## 方法总结

音频只是二进制的载体，决定性步骤是稳定转写并保持字节边界。长位流适合先抽查几个字节能否形成英文，再自动完成剩余分组，避免手工计数产生级联错位。
