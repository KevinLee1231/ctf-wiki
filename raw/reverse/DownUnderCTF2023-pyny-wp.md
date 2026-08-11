# DownUnderCTF 2023 pyny Writeup

## 题目简述

附件看起来是普通 Python 文件，但源码开头声明了 `# coding: punycode`，后续标识符和语句因而呈现为难以直接阅读的 Unicode 形式。题目的核心是 Python 源码编码声明和 Punycode 解码，而不是破解某种加密算法。

## 解题过程

Python 会按照首两行中的编码声明解码整个源文件。为了观察解码后的真实源码，先去掉编码声明，再把剩余原始字节按 Punycode 解码。要保留声明之后的换行，否则解码结果的边界可能发生变化。

```python
from pathlib import Path

raw = Path("pyny.py").read_bytes()
body = raw.split(b"\n", 1)[1]
print(body.decode("punycode"))
```

恢复后的代码会显露原本被编码隐藏的属性名和字符串，其中给出了 flag 的主体 `python_warmup`。按比赛格式补全后得到：

```text
DUCTF{python_warmup}
```

## 方法总结

Python 的 `coding` 声明不仅影响字符串，也决定解释器如何读取整个源文件。分析此类题目时应保留原始字节，先确认编码边界，再使用对应 codec 还原源码；直接复制终端中已经错误解码的字符，容易造成不可逆的信息损失。
