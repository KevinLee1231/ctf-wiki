# Satellite

## 题目简述

题目给出一大段外观相近的 Unicode 字符。绝大多数是西里尔字母，真正信息混在其中的 ASCII 字母、数字和 flag 标点里，属于同形异码字符混淆。

## 解题过程

肉眼看起来相同的 `A`、`a`、`e` 等字符可能属于完全不同的 Unicode 区段。保留 ASCII 范围内可能出现在 flag 中的字符，丢弃其余字符：

```python
import re

text = open("satellite.txt", encoding="utf-8").read()
plain = "".join(re.findall(r"[A-Za-z0-9_{}$!?@-]", text))
print(plain)
```

输出直接形成：

```text
UMDCTF-{RIP_sPutN1k_1962}
```

## 方法总结

Unicode 隐写首先要比较码点，而不是比较字形。可用 `ord()`、Unicode 名称或字符区段快速区分 ASCII 与西里尔同形字。本题有效字符本身已包含完整格式，过滤时应白名单保留大括号、下划线等符号，避免只留字母数字后又猜测结构。
