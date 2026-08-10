# Sign In Pro Max

## 题目简述

题目把 flag 拆成五段，分别用多层 Base 编码、MD5、SHA-1、SHA-256 和凯撒移位隐藏。最终需要按 UUID 的 `8-4-4-4-12` 长度格式拼接五段结果。

原始线索为：

```text
Part1: QVl5Y3BNQjE1ektibnU3SnN6M0tGaQ==
Part2: c629d83ff9804fb62202e90b0945a323
Part3: 99f3b3ada2b4675c518ff23cbd9539da05e2f1f8
Part4: 1838f8d5b547c012404e53a9d8c76c56399507a2b017058ec7f27428fda5e7db
Part5: Ufwy5 nx 0gh0jf61i21h, stb uzy fqq ymj ufwyx ytljymjw, its'y ktwljy ymj ktwrfy.
```

## 解题过程

第一段结尾有 `==`，先按 Base64 解码：

```text
AYycpMB15zKbnu7Jsz3KFi
```

这个字符串只使用 Base58 字符表，继续解码得到：

```text
MY2TCZBTMEYTQ===
```

再按 Base32 解码，得到第一段 `f51d3a18`。

第二段是 128 位摘要且分组长度为 512 位，对应 MD5；第三段是 160 位摘要，对应 SHA-1。题目中的原文都很短，可以查表或枚举四位十六进制字符串，分别得到：

```text
MD5  -> f91c
SHA1 -> 4952
```

第四段是 SHA-256。根据最终 UUID 格式，这一段同样只有 4 个字符；对四位字母数字串进行枚举并比较摘要，可恢复 `a3ed`。三个摘要可以直接校验：

```python
import hashlib

assert hashlib.md5(b"f91c").hexdigest() == "c629d83ff9804fb62202e90b0945a323"
assert hashlib.sha1(b"4952").hexdigest() == "99f3b3ada2b4675c518ff23cbd9539da05e2f1f8"
assert hashlib.sha256(b"a3ed").hexdigest() == (
    "1838f8d5b547c012404e53a9d8c76c56399507a2b017058ec7f27428fda5e7db"
)
```

第五段是字母向后偏移 5 位的凯撒密码。将每个字母回退 5 位后得到：

```text
Part5 is 0bc0ea61d21c, now put all the parts together, don't forget the format.
```

按 `8-4-4-4-12` 拼接：

```text
hgame{f51d3a18-f91c-4952-a3ed-0bc0ea61d21c}
```

## 方法总结

识别编码时可以从字符表、填充符和长度入手：Base64 常见 `+`、`/` 与最多两个 `=`，Base58 排除易混淆字符，Base32 使用大写字母和 `2` 至 `7`。摘要不能直接“解密”，但当明文空间被格式和长度限制得很小时，可以枚举候选并重新计算摘要验证。
