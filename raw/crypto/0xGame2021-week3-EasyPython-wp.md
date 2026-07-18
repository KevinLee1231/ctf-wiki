# week3EasyPython

## 题目简述

官方 WP 将本题描述为“8 进制转字符串，2020 纵横杯签到题”。核心是把一组八进制 ASCII 数值还原为字符。

## 解题过程

若附件形如以空格分隔的 `60 170 107 ...`，逐项按八进制转整数，再调用 `chr`：

```python
from pathlib import Path

tokens = Path("cipher.txt").read_text(encoding="utf-8").split()
plain = "".join(chr(int(token.removeprefix("0o"), 8)) for token in tokens)
print(plain)
```

如果附件使用 Python 风格的八进制转义，如 `\060\170\107...`，可安全地把它作为字符串字面量解析：

```python
import ast
from pathlib import Path

escaped = Path("cipher.txt").read_text(encoding="utf-8").strip()
plain = ast.literal_eval(repr(escaped).replace("\\\\", "\\"))
print(plain)
```

当前仓库没有本题目录、密文附件或最终 flag；官方 Week 3 PDF 对照页也只有上述一句说明。因此可以确认八进制 ASCII 机制，但无法从现存材料准确恢复原始密文和答案。

## 方法总结

八进制编码常见为数值列表或 `\ooo` 转义序列，两种格式要分别处理。转换前先确认表示形式，并在附件缺失时明确证据边界，不把同名签到题的网络答案当作本题结果。
