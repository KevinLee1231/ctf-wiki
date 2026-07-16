# Code-Vigenere

## 题目简述

题目把 flag 作为 Vigenère 明文，加密时仅对英文字母应用循环移位，数字、连字符和花括号原样保留，而且这些非字母不消耗密钥位置。密钥由随机字节经 Base64 后截取前 5 个字符得到，因此有效密钥周期为 5。

```python
key = b64encode(urandom(6))[:5].decode()
print(Encrypt(flag, key))
```

给出的密文为：

```text
0lCcop{oyd94092-g8mq-4963-88b6-4helrxdhm6q7}
```

## 解题过程

flag 的已知前缀是 `0xGame{`。将其中五个字母 `xGame` 与密文对应字母 `lCcop` 相减，即可恢复一个完整周期的有效移位：

```text
plaintext:  x G a m e
ciphertext: l C c o p
key shift:  O W C C L
```

这里恢复的是模 26 的有效移位；即使原始 Base64 密钥字符包含数字或符号，也不影响用这组移位解密。

完整求解脚本如下：

```python
cipher = "0lCcop{oyd94092-g8mq-4963-88b6-4helrxdhm6q7}"
known = "0xGame{"

key = "".join(
    chr((ord(c.upper()) - ord(p.upper())) % 26 + ord("A"))
    for p, c in zip(known, cipher)
    if p.isalpha()
)

def decrypt(text, key):
    shifts = [ord(ch) - ord("A") for ch in key]
    result = []
    index = 0
    for ch in text:
        if not ch.isalpha():
            result.append(ch)
            continue
        base = ord("A") if ch.isupper() else ord("a")
        result.append(chr(base + (ord(ch) - base - shifts[index % len(key)]) % 26))
        index += 1
    return "".join(result)

print(key)
print(decrypt(cipher, key))
```

输出为：

```text
OWCCL
0xGame{acb94092-e8bc-4963-88f6-4fcadbbfb6c7}
```

## 方法总结

- 核心技巧：利用已知 flag 前缀实施 Vigenère 已知明文攻击，恢复完整密钥周期。
- 识别信号：字母被周期性替换，而数字、标点和格式保持不变；已知明文长度覆盖密钥周期。
- 复用要点：必须与题目实现保持同样的密钥索引规则，本题遇到非字母时不递增索引，否则后续字符会全部错位。
