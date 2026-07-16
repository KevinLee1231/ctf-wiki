# Vigenere

## 题目简述

题目把经典 Vigenere 密码扩展到 `digits + ascii_letters + punctuation` 组成的 94 字符字母表。密钥 `Welcome-2025-0xGame` 已知并循环使用，每个明文字节在该字母表中的索引与密钥索引相加后取模。

## 解题过程

设字母表长度为 $N=94$，加密关系为 $C_i=(P_i+K_i)\bmod N$，因此逐字符计算 $P_i=(C_i-K_i)\bmod N$ 即可逆转。这里的“减法”是索引运算，不能直接对字符编码值相减。

```python
from string import digits, ascii_letters, punctuation

key = "Welcome-2025-0xGame"
alphabet = digits + ascii_letters + punctuation

def vigenere_decrypt(ciphertext, key):
    plaintext = ""
    key_index = 0
    for char in ciphertext:
        bias = alphabet.index(key[key_index])
        char_index = alphabet.index(char)
        new_index = (char_index - bias) % len(alphabet)
        plaintext += alphabet[new_index]
        key_index = (key_index + 1) % len(key)
    return plaintext

ciphertext = r'WL"mKAaequ{q_aY$oz8`wBqLAF_{cku|eYAczt!pmoqAh+'
print(vigenere_decrypt(ciphertext, key))
```

原密文含引号、反斜线等标点，使用原始字符串可避免 Python 转义改变数据。运行脚本后直接得到符合 `0xGame{...}` 格式的明文。

## 方法总结

- 核心技巧：在自定义字母表上逆转已知密钥的 Vigenere 索引加法。
- 识别信号：密钥循环、字符索引相加取模，以及固定可打印字母表。
- 复用要点：先完整复刻字母表及顺序；字母表顺序、密钥索引推进或字符串转义任一不一致都会导致全局解密错误。
