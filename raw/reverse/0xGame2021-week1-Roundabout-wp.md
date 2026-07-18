# week1Roundabout

## 题目简述

脱壳后可见程序用固定字符串 `this_is_not_flag` 作为 16 字节循环密钥，与 42 个密文字节逐项异或。异或可逆，重复同一操作即可恢复 flag。

## 解题过程

定位校验循环后，可以把关系写成：

$$
plain_i=cipher_i\oplus key_{i\bmod16}
$$

源码字符串后的 `\x00` 只是 C 字符串终止符，不参与 `i % 16` 的密钥循环。恢复代码如下：

```python
cipher = [
    68, 16, 46, 18, 50, 12, 8, 61, 86, 10, 16, 103, 0, 65, 0, 1,
    70, 90, 68, 66, 110, 12, 68, 114, 12, 13, 64, 62, 75, 95, 2, 1,
    76, 94, 91, 23, 110, 12, 22, 104, 91, 18,
]
key = b"this_is_not_flag"

flag = bytes(
    value ^ key[index % len(key)]
    for index, value in enumerate(cipher)
)
print(flag.decode())
```

输出为：

```text
0xGame{b8ed8f-af22-11e7-bb4a-3cf862d1ee75}
```

## 方法总结

本题的关键是脱壳后识别固定密钥和循环索引。异或链本身没有单向性；只要保留密文、密钥以及周期 16，便能完整离线恢复结果。
