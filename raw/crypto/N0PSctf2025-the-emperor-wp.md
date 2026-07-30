# N0PSctf2025 The Emperor Writeup

## 题目简述

题目给出一段只由英文字母和标点组成的密文，并提示这是面向新手的古典密码题。flag 格式为 `B4BY{decoded message}`，因此目标是识别字母替换规律并恢复大写明文。

## 解题过程

对密文尝试固定字母位移后，可以发现它是移位量为 12 的 Caesar 密码。解密时把每个字母在字母表中的编号减去 12，即：

$$
p=(c-12)\bmod 26
$$

可以用下面的脚本直接恢复明文：

```python
ciphertext = """
Ea, kag pqoapqp uf tgt?
Ftqz, tqdq ue ftq rxms:
UVGEFPQOAPQPMOMQEMDOUBTQDITUOTYMWQEYQMBDARQEEUAZMXODKBFATQDA
"""

result = []
for ch in ciphertext:
    if ch.isalpha():
        base = ord("A") if ch.isupper() else ord("a")
        result.append(chr((ord(ch) - base - 12) % 26 + base))
    else:
        result.append(ch)

print("".join(result))
```

最后一行解出：

```text
IJUSTDECODEDACAESARCIPHERWHICHMAKESMEAPROFESSIONALCRYPTOHERO
```

因此 flag 为：

```text
B4BY{IJUSTDECODEDACAESARCIPHERWHICHMAKESMEAPROFESSIONALCRYPTOHERO}
```

## 方法总结

- 核心技巧：枚举或根据可读片段判断 Caesar 固定位移。
- 识别信号：密文保留空格和标点，字母频率与单词长度结构接近自然语言。
- 复用要点：解密位移应取加密位移的相反数，并分别保持大小写字母的取模范围。
