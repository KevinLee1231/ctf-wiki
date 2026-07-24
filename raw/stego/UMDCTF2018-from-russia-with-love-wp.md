# UMDCTF 2018 - From Russia With Love

## 题目简述

附件 `comrade.txt` 表面是一封英文电报，但其中混入了全角拉丁字母、希腊字母、西里尔字母、罗马数字字符和 Unicode 连字符。它们在视觉上与普通 ASCII 字符相近，构成了同形异码字符隐写。

## 解题过程

先检查每个字符是否属于 ASCII。仓库中的 `dev/decode.rb` 已给出最小提取逻辑：只输出 `ascii_only?` 为假的字符。等价的 Python 写法是：

```python
from pathlib import Path

text = Path("comrade.txt").read_text(encoding="utf-8")
hidden = "".join(char for char in text if not char.isascii())
print(hidden)
```

提取结果为：

```text
ＵМⅮⅭΤＦ‐ΑТΑⅭКАＴⅮАＷＮ
```

其中既有全角字符，也有外形近似的 Greek Alpha、Cyrillic Em、Roman Numeral Five Hundred 等字符。不能只依赖 NFKC：它能把全角字母和罗马数字转为 ASCII，却不会把希腊字母或西里尔字母自动改成外形相同的拉丁字母。按字形逐一归一化后得到：

```text
UMDCTF-ATTACKATDAWN
```

这道题的 flag 没有花括号。其 SHA-256 为：

```text
c2e21cd023b598f7ab33965a4e6150a67944ea6b163f7fd5abae18b93873a48f
```

与 `README.md` 中的官方摘要完全一致。

## 方法总结

同形异码隐写的证据不在“显示出来像什么”，而在字符的实际码点。应先按 ASCII 与非 ASCII 分流，再结合 Unicode 名称和字形归一化；不要擅自补充常见 flag 格式，最终形式应由附件与摘要共同确认。
