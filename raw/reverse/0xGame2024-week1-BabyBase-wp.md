# BabyBase

## 题目简述

程序会对输入执行一次编码，再由 `check_flag()` 比较编码结果。二进制中同时保存了标准 Base64 索引表和目标密文，因此只需识别编码算法并逆向解码。

## 解题过程

用 IDA 反编译主函数，可以看到与校验相关的 `encode()` 和 `check_flag()` 两个函数。

Strings 窗口中出现了标准 Base64 索引表：

```text
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
```

继续分析 `encode()`，可以观察到“每 3 字节输入转换为 4 个字符”以及末尾使用 `=` 补齐等特征，确认它实现的是标准 Base64 编码。

`check_flag()` 使用的目标密文为：

```text
MHhHYW1le04wd195MHVfa24wd19CNHNlNjRfRW5jMGQxbmdfdzNsbCF9
```

直接用 Python 标准库解码：

```python
import base64

ciphertext = b"MHhHYW1le04wd195MHVfa24wd19CNHNlNjRfRW5jMGQxbmdfdzNsbCF9"
print(base64.b64decode(ciphertext).decode())
```

输出为：

```text
0xGame{N0w_y0u_kn0w_B4se64_Enc0d1ng_w3ll!}
```

## 方法总结

Base64 的典型识别特征包括 64 字符索引表、3 字节到 4 字符的分组转换和 `=` 填充。确定算法后，从校验函数提取目标字符串并执行标准 Base64 解码即可。
