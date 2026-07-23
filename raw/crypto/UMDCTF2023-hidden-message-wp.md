# Hidden Message

## 题目简述

附件 `text.txt` 看似由宝可梦相关的普通单词组成，但每个单词内部大小写混杂，例如：

```text
GO CATCH tHe bug parAS! AbRa LIKES His speED!
```

题面提示“分隔符是空格”。这里的单词内容只是载体，真正的数据是每个字母的大小写模式；空格将载体分成一个个 Morse 字符。

## 解题过程

把大写字母映射为划 `-`，小写字母映射为点 `.`，忽略标点。例如：

```text
GO     -> --    -> M
CATCH  -> ----- -> 0
tHe    -> .-.   -> R
bug    -> ...   -> S
parAS  -> ...-- -> 3
```

于是第一组单词解码为 `M0RS3`。文本中的感叹号把若干 Morse 字符组分隔成明文单词，完整结果依次是：

```text
M0RS3 C0D3 M34NS H4V1NG S0 M8CH F8NS13S
```

一个简化解码器如下：

```python
import re

MORSE = {
    # 标准 Morse 字母与数字表
}

groups = open("text.txt", encoding="utf-8").read().split("!")
decoded_groups = []

for group in groups:
    chars = []
    for token in group.split():
        letters = re.sub(r"[^A-Za-z]", "", token)
        code = "".join("-" if c.isupper() else "." for c in letters)
        chars.append(MORSE[code])
    if chars:
        decoded_groups.append("".join(chars))

print(" ".join(decoded_groups))
```

按赛事 flag 格式补回固定前缀和花括号，注意正文空格也是 flag 的一部分：

```text
UMDCTF{m0rs3 c0d3 m34ns h4v1ng s0 m8ch f8ns13s}
```

## 方法总结

- 核心技巧：忽略单词语义，把字母大小写分别视为 Morse 的划与点，以空格分隔字符、感叹号分隔明文单词。
- 识别信号：自然语言载体中存在刻意而稳定的大小写混排，且各 token 长度集中在 1 至 5 时，应检查 Morse 或其他二元码。
- 复用要点：先验证几个 token 是否能映射成连贯字符，再确定点划方向；标点通常承担更高一层的分组功能，不应直接混入 Morse 码。
