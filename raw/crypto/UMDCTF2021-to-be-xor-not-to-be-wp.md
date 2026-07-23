# To Be Xor Not To Be

## 题目简述

题目给出一个二进制位串和密钥文本 `this is the key!`。关键在于还原出题代码的数值表示：两者被解释为大端整数后整体异或，而不是常见的逐字节循环密钥 XOR。

## 解题过程

将密文位串直接转为整数，将密钥文本按大端字节序转为整数：

```python
c = int(ciphertext_bits, 2)
k = int.from_bytes(b"this is the key!", "big")
m = c ^ k
plain = m.to_bytes((m.bit_length() + 7) // 8, "big")
print(plain.decode())
```

大整数异或会按最低有效位对齐两个操作数，因此不需要人为补齐或重复密钥。输出为：

```text
UMDCTF-{w3lc0m3_t0_crypt0}
```

## 方法总结

“XOR”只说明运算，不说明数据布局。位串转整数、固定长度字节串异或和循环密钥异或会产生不同对齐方式。遇到题目源码时应完全复现其类型转换；本题只需一次大整数异或，任何重复密钥步骤都会引入错误。
