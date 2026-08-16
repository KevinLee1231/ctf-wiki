# Kindile_eBook_1

## 题目简述

附件是一段来自英文书籍的长文本，但每个英文字母都被固定替换成另一个字母，属于单表替换密码。题名和题面提到 Al-Kindi，也在提示经典频率分析。文本足够长，重复单词和自然语言结构能逐步确定替换表。

## 解题过程

### 从单词模式建立初始映射

密文中 `boeb` 多次出现，模式为 `ABCA`。结合上下文，它很可能对应常见词 `that`，于是先得到：

```text
b -> t
o -> h
e -> a
```

把已知映射代回全文后，会逐渐显出诸如 `the`、`people`、`learning` 等词。再利用单词长度、重复字母位置和上下文补齐其他字母。单表替换要求映射一一对应；若一个猜测使两个密文字母映射到同一明文字母，就应回退。

### 用频率分析辅助人工修正

可先统计字母出现次数，优先尝试将高频密文字母对应到英文中的 `e`、`t`、`a`、`o`、`i`、`n`：

```python
from collections import Counter

ciphertext = open("Kindile_eBook_1", encoding="utf-8").read().lower()
frequency = Counter(ch for ch in ciphertext if ch.isalpha())
print(frequency.most_common())
```

频率只能给候选，不能机械按排名一一替换；例如 `th`、`he`、`ing` 等二元和三元组合，以及 `ABCA` 这类单词形状，通常比单字母频率更可靠。完成替换后，正文中的提示句还原为：

```text
you can have your flag now shellmates{JustASimpleAnalysis}
```

flag 为：

```text
shellmates{JustASimpleAnalysis}
```

## 方法总结

- 核心技巧：结合字母频率、重复单词形状和英文上下文破解单表替换。
- 识别信号：长篇自然语言中字符间保持一一对应、空格和标点未被破坏，是古典替换密码的典型特征。
- 复用要点：频率分析只负责缩小候选；最终映射必须通过词形、上下文和一一对应约束交叉验证。
