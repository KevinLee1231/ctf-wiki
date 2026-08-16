# Kindile_eBook_2

## 题目简述

本题继续使用单表替换，但密文字母被替换为数字和标点符号。附件仍是一段很长的英文说明，内容正是 Al-Kindi 对频率分析的描述。由于符号种类有限且每个符号固定代表一个字母，可以通过英文字符频率和上下文恢复明文。

## 解题过程

### 统计符号频率

先忽略空白，统计所有替换符号的出现次数：

```python
from collections import Counter

text = open("Kindile_eBook_2", encoding="utf-8").read()
counts = Counter(ch for ch in text if not ch.isspace())
for symbol, count in counts.most_common():
    print(repr(symbol), count)
```

官方分析中，最高频的若干对应关系为：

```text
# -> e    ] -> t    1 -> a    + -> o
~ -> n    [ -> i    & -> h    | -> s
^ -> r    ) -> l    ! -> d    @ -> u
```

这些只是初始映射。频率相近的字母可能交换，例如 `h` 与 `s` 的英文总体频率接近，仍需用单词结构和上下文判断。

### 逐步代换并检查自然语言

将已确定的符号替换为字母，未确定符号保留原样：

```python
mapping = {
    "#": "e", "]": "t", "1": "a", "+": "o",
    "~": "n", "[": "i", "&": "h", "|": "s",
    "^": "r", ")": "l", "!": "d", "@": "u",
}

preview = "".join(mapping.get(ch, ch) for ch in text)
print(preview)
```

利用已经显现的 `the`、`cryptanalysis`、`message` 等词继续补全映射，最终能读到：

```text
oh cool another flag here shellmates{CryptanalysisCanDoSoooMuch}
```

得到 flag：

```text
shellmates{CryptanalysisCanDoSoooMuch}
```

## 方法总结

- 核心技巧：把符号视作替代字母，先按频率建立候选表，再用自然语言结构校正。
- 识别信号：符号集规模接近字母表、相同符号在全文含义固定且文本很长时，应考虑单表替换而非复杂编码。
- 复用要点：不要只按频率排名强行一对一匹配；使用常见词、重复模式和相邻字母频率能显著减少误判。
