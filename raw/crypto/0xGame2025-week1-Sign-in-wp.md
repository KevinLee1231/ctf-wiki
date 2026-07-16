# Sign_in

## 题目简述

附件是一段 Base64 文本。解码后得到的仍不是 flag，而是经过凯撒移位的可打印字符串；已知 flag 前缀为 `0xGame`，可以用它筛选正确移位量。

## 解题过程

先进行 Base64 解码，再遍历凯撒密码的 26 种位移。字母分别在大小写字母表内循环，数字、花括号和下划线保持不变。当结果以 `0xGame` 开头时，即得到唯一合理明文。

```python
import base64

def caesar_decrypt(text, shift):
    result = ""
    for char in text:
        if char.isupper():
            result += chr((ord(char) - 65 - shift) % 26 + 65)
        elif char.islower():
            result += chr((ord(char) - 97 - shift) % 26 + 97)
        else:
            result += char
    return result

def solve(ciphertext):
    decoded = base64.b64decode(ciphertext).decode("utf-8")
    for shift in range(26):
        plaintext = caesar_decrypt(decoded, shift)
        if plaintext.startswith("0xGame{"):
            print(plaintext)
            break

solve("MGhRa3dve0dvdm0wd29fZDBfMGhRNHczXzJ5MjVfQHhuX3JAbXVfUHliX3BlWH0=")
```

## 方法总结

- 核心技巧：按编码层次先 Base64、后凯撒移位，并利用已知格式筛选结果。
- 识别信号：字符串字符集和末尾填充符合 Base64；解码结果可打印但不可读，且字母分布像单表移位。
- 复用要点：多层编码题应每解一层就检查数据形态，不要把 Base64 当作加密；凯撒密码只有 26 种密钥，穷举比猜偏移更可靠。
