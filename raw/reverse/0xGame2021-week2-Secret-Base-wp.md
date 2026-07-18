# week2Secret Base

## 题目简述

程序实现了标准 Base64 的 6 位分组过程，但把索引到字符的字母表替换为自定义顺序。密文和自定义表都硬编码在程序中，只需先把字符映射回标准 Base64 字母表，再调用常规解码器。

## 解题过程

标准表与题目表分别为：

```text
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
123DfgabcQeEFh4pklmnojqGHIJKLMNOPdRSTUVWXYZrstuvwxyz0ABCi56789+/
```

编码函数仍然每 3 字节拆成 4 个 6 位索引，并使用 `=` 补齐，所以变更的只有“索引对应哪个字符”。对密文逐字符执行 `custom[index] → standard[index]` 的翻译，再做标准 Base64 解码：

```python
import base64

ciphertext = "FbdbHqAUNzoiIDdUIDhSEnF54DHthDT5Hm05HVlREnoAhaF0Hn2dFBQVFC0="
standard = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
custom = "123DfgabcQeEFh4pklmnojqGHIJKLMNOPdRSTUVWXYZrstuvwxyz0ABCi56789+/"

translated = ciphertext.translate(str.maketrans(custom, standard))
print(base64.b64decode(translated).decode())
```

输出为：

```text
0xGame{58d8ed3c-3986-499a-9bdb-554c4a0a3bf3}
```

## 方法总结

改表 Base64 的位分组和补位规则没有改变，变化的只是 64 个符号的排列。恢复时应建立按相同索引对应的翻译表，而不是把自定义表排序或直接对密文调用标准解码。
