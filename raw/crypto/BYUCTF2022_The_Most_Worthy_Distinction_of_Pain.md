# BYUCTF 2022 - The Most Worthy Distinction of Pain

## 题目简述

附件 `encrypted.txt` 是一串主要以 `d` 开头的英文单词，并给出了 Go 加密源码。源码依赖 Project Gutenberg 的 `CROSSWD.TXT` 词表：文件 MD5 应为 `e58eb7b851c2e78770b20c715d8f8d7b`，第 1 行为 `aa`，第 113809 行为 `zymurgy`。

## 解题过程

阅读 `encrypt.go` 可知，明文每两个字节按大端序合成一个 `uint16`：

```text
index = high_byte << 8 | low_byte
```

随后程序从词表中取第 `index` 行作为密文单词。解密只需执行逆过程：查出每个单词在词表中的一基下标，再拆成高、低两个字节。官方 `decrypt.go` 的核心等价于：

```go
value := lineNumber // 第一行编号为 1
plain = append(plain, byte(value>>8), byte(value&0xff))
```

仓库未附带大词表，因此需下载题面指定的 [CROSSWD.TXT](https://www.gutenberg.org/files/3201/files/CROSSWD.TXT)，并先核对哈希与首尾行，避免因词表版本或换行差异导致所有索引错位。按顺序解码 `encrypted.txt` 后，明文中出现：

```text
byuctf{what_an_inefficient_code_ug2Ko8Cz}
```

## 方法总结

这是“词表索引编码”，不是根据单词语义做密码分析。源码已经完整定义了字节序、一基下标和词表版本；复现时最容易出错的是把索引当成零基，或使用内容不同的词表。
