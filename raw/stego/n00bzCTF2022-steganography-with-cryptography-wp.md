# stegnography_with_cryptography

## 题目简述

两张 JPEG 分别通过 `steghide` 藏入一份密钥和一份密文。第一层需要用字典恢复嵌入文件，第二层再做重复密钥异或。

## 解题过程

分别用 `rockyou.txt` 对两张图执行 `stegseek`：

```bash
stegseek 1.jpg /usr/share/wordlists/rockyou.txt
stegseek 2.jpg /usr/share/wordlists/rockyou.txt
```

得到的 `a.txt` 内容是 ASCII 字符串 `0xdeadbeef`，`b.txt` 则是不可打印的二进制密文。这里不能把 `0xdeadbeef` 当成四字节整数；官方逻辑是重复使用它的十个 ASCII 字节：

```python
from itertools import cycle

key = b"0xdeadbeef"
enc = open("b.txt", "rb").read()
plain = bytes(a ^ b for a, b in zip(enc, cycle(key)))
print(plain.decode())
```

输出为：

```text
n00bz{st3gn0gr4phy_w1th_crypt0gr4phy_1s_fun!}
```

## 方法总结

本题的两层证据边界很清晰：先恢复嵌入文件，再依据文件的真实字节表示解密。看到十六进制外观的字符串时，必须确认它是文本密钥还是应解析为数值，否则会得到完全不同的密钥流。
