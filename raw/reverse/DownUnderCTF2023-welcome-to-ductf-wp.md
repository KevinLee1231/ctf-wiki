# DownUnderCTF 2023 Welcome to DUCTF Writeup

## 题目简述

附件是一份上下颠倒的 Aussie++ 源码。字符不仅排列方向反转，部分字母和符号也被替换成对应的倒置 Unicode 字形。目标是先恢复可读源码，再定位真正参与 flag 拼接的变量。

## 解题过程

恢复过程分为两步：先倒转整份文件的行序，并逐行反转字符；再按成对的正立、倒立字符表进行替换。例如 `a` 与 `ɐ`、`b` 与 `q`、`<` 与 `>` 需要互相映射。

官方解法的核心可以整理为：

```python
from pathlib import Path

flips = {
    "ɐ": "a", "q": "b", "ɔ": "c", "p": "d", "ǝ": "e",
    "ɟ": "f", "ƃ": "g", "ɥ": "h", "ᴉ": "i", "ɾ": "j",
    "ʞ": "k", "ɯ": "m", "u": "n", "ɹ": "r", "ʇ": "t",
    "ʌ": "v", "ʍ": "w", "ʎ": "y", "<": ">", ">": "<",
    "(": ")", ")": "(", "¿": "?", "¡": "!",
}

lines = Path("upside-down.a++").read_text(encoding="utf-8").splitlines()
text = "\n".join(line[::-1] for line in lines[::-1])
print("".join(flips.get(char, char) for char in text))
```

完整字符表还包含其余大小写字母和数字；按相同的成对映射补齐即可。恢复后的 Aussie++ 程序中，`GLAF` 保存前缀 `DUCTF{`，`STREWTH` 保存右花括号，而变量 `HappyHour` 的值为：

```text
1ts-5oCl0cK_5om3wh3RE
```

三部分拼接得到：

```text
DUCTF{1ts-5oCl0cK_5om3wh3RE}
```

## 方法总结

这道题同时使用了文本方向反转和 Unicode 字形替换。处理此类源码时，应先恢复空间结构，再恢复字符本身；只做其中一步仍会得到不可读文本。最终无需依赖在线解释器，阅读还原后的变量赋值和字符串拼接就能闭合解题证据。
