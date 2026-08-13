# Flag Checker Part 999

## 题目简述

题目给出一个 64 位 flag checker。程序内置一段密文字节和循环密钥 `mAn_whO_aM_i`，检查逻辑只是把两者按周期逐字节异或，并与用户输入比较。

## 解题过程

从反编译结果提取 47 字节数组和密钥。异或具有自反性，相同密钥既用于加密也用于解密；当密钥短于密文时按周期重复即可：

```python
encrypted = bytes([
    10, 51, 11, 38, 12, 16, 127, 45, 62, 33, 111, 89, 29, 116, 49,
    62, 37, 13, 16, 53, 52, 62, 43, 54, 25, 9, 93, 0, 21, 13, 8,
    54, 15, 35, 54, 7, 10, 30, 6, 56, 29, 14, 36, 44, 5, 48,
])
key = b"mAn_whO_aM_i"

flag = bytes(
    value ^ key[index % len(key)]
    for index, value in enumerate(encrypted)
)
print(flag.decode())
```

输出为：

```text
grey{x0r_l00p5_aRe_jUst_tH3_beGinning_hgjfksd}
```

将该字符串输入原始 `flagchecker`，程序实际返回 `Right password!`，说明数组长度、密钥周期和字符顺序均恢复正确。

## 方法总结

固定密钥循环异或不提供密码学安全性；只要密钥和密文都嵌在程序中，就能直接恢复明文。分析时应以密文长度控制循环，避免因 C 字符串终止符或密钥长度误判而多解一字节。最终 flag 为 `grey{x0r_l00p5_aRe_jUst_tH3_beGinning_hgjfksd}`。
