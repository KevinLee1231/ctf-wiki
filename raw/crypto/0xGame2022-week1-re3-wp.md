# week1re3

## 题目简述

程序中保存了一段使用标准 Base64 编码的字符串，没有额外的自定义变换或加密。

## 解题过程

字符串只由 Base64 字母表字符组成，长度是 4 的倍数，末尾也符合 Base64 的填充特征。直接用 Python 标准库解码：

~~~python
import base64

encoded = b"ZmxhZ3tkYWJkZDMzMC0yN2RiLTRlOGEtYjBjZi0zMTAzMDVlYzAzNWJ9"
print(base64.b64decode(encoded).decode())
~~~

得到：

```text
flag{dabdd330-27db-4e8a-b0cf-310305ec035b}
```

## 方法总结

- 核心方法：识别标准 Base64 字符串并直接解码。
- 识别特征：字符集集中在 `A-Z`、`a-z`、`0-9`、`+`、`/`，长度通常是 4 的倍数，并可能以 `=` 填充。
- 注意事项：Base64 是编码而不是加密；解码后仍应检查结果是否满足题目要求的 flag 格式。
