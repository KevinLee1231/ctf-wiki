# Height's Hidden History

## 题目简述

附件是一段普通历史文本，其中少量字符被波浪号 `~` 替换。生成脚本先对 flag 做 Base64，再依次寻找每个 Base64 字符在原文中的首次可用位置并替换，因此被遮住字符的顺序就是编码后的 flag。

## 解题过程

如果同时拥有题目文本与仓库中的原始模板，就逐字符比较：题目中为 `~` 的位置，对应原模板字符就是被隐藏的 Base64 字符。保持文本顺序收集后解码：

```python
import base64
from pathlib import Path

original = Path("original.txt").read_text()
hidden = Path("history.txt").read_text()
assert len(original) == len(hidden)

encoded = "".join(
    source
    for source, current in zip(original, hidden)
    if current == "~"
)
print(base64.b64decode(encoded).decode())
```

得到：

```text
UMDCTF-{shr3w_l3tt3r}
```

## 方法总结

占位符隐写的核心是位置和原始载体之间的差分。不要把 `~` 本身当作摩斯码；它只标记“这里原来有一个有意义字符”，恢复字符序列后还需按 Base64 解码。
