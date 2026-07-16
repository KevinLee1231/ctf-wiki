# 8848

## 题目简述

服务端用一个超过 64 字节的长口令创建 WinZip AES 加密 ZIP，却只允许输入不超过 30 个字符的 Base64 文本。WinZip AES 的密钥派生使用 PBKDF2-HMAC-SHA1；HMAC-SHA1 会先把超过 64 字节块长的密钥压缩为 20 字节 SHA-1 摘要，因此“原长口令”和“该口令的原始 SHA-1 摘要”会导出相同密钥。

## 解题过程

源码中的原始口令是：

```text
very_very_very_very_long_password_which_cannot_be_cracked_easily_and_will_never_be_known_to_anyone
```

计算其 SHA-1 原始摘要，再做 Base64 编码，使它能够通过文本输入传给服务端：

```python
from base64 import b64encode
from hashlib import sha1

password = (
    b"very_very_very_very_long_password_which_cannot_be_cracked_"
    b"easily_and_will_never_be_known_to_anyone"
)

digest = sha1(password).digest()
answer = b64encode(digest)
print(digest.hex())
print(answer.decode(), len(answer))
```

输出为：

```text
481d393bea2692b07dbd4633cafaea0352d266d8
SB05O+omkrB9vUYzyvrqA1LSZtg= 28
```

28 字符满足 `len(text) <= 30`。服务端执行 `base64.b64decode()` 后得到 20 字节 SHA-1 摘要，并把它作为 pyzipper 解压口令；HMAC 的长密钥规范化使其与原口令等价，因此成功解压并输出：

```text
0xGame{B07h_z1p_&_8848_Can_h4v3_Two_P@ssw0rds}
```

## 方法总结

这不是 Base64 压缩，也不是暴力破解，而是 HMAC 对超块长密钥的标准化行为造成的等价表示。审计密码长度限制时必须追踪输入在每一层的真实字节形式；只限制编码后字符串长度，未必限制最终送入密钥派生函数的有效秘密。
