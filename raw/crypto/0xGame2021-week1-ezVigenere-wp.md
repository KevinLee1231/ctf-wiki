# week1ezVigenere

## 题目简述

附件是一段维吉尼亚密文，末尾仍能看出 flag 的轮廓。利用已知格式 `0xGame{` 可以恢复短密钥并解密全文。

## 解题过程

维吉尼亚密码满足 $C_i\equiv P_i+K_i\pmod{26}$。将密文末尾的 `0yIang{` 与预期明文 `0xGame{` 对齐，对字母逐位计算 $K_i=C_i-P_i\pmod{26}$，得到周期密钥 `abc`。数字和标点不参与密钥下标推进。

```python
from pathlib import Path

ciphertext = Path("ezVigenere.txt").read_text(encoding="utf-8")
key = "abc"
plain = []
j = 0

for ch in ciphertext:
    if ch.isascii() and ch.isalpha():
        base = ord("A") if ch.isupper() else ord("a")
        shift = ord(key[j % len(key)]) - ord("a")
        plain.append(chr((ord(ch) - base - shift) % 26 + base))
        j += 1
    else:
        plain.append(ch)

text = "".join(plain)
print(text[text.index("0xGame{"):])
```

得到：

```text
0xGame{interest1ng_Vigenere}
```

## 方法总结

已知明文格式可以把维吉尼亚密码转化为已知明文攻击：用对应字符之差恢复密钥周期，再本地解密。这样不需要依赖词频分析网站，也不会受在线工具默认参数影响。
