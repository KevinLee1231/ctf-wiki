# Vinegar2

## 题目简述

第二版把维吉尼亚码表扩展到大小写字母、数字和特殊字符。密钥与密文等长，源码给出了完整字符表和加密方式。

## 解题过程

字符表为：

```text
abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890!@#$%^&*(){}_?
```

加密本质仍是索引相加，所以解密按字符表长度 $N$ 做索引相减：

```python
alphabet = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890!@#$%^&*(){}_?"
ciphertext = "*fa4Q(}$ryHGswGPYhOC{C{1)&_vOpHpc2r0({"
key = "5up3r_s3cr3t_k3y_f0r_1337h4x0rs_r1gh7?"

plain = "".join(
    alphabet[(alphabet.index(c) - alphabet.index(k)) % len(alphabet)]
    for c, k in zip(ciphertext, key)
)
print(plain)
```

输出：

```text
n00bz{4lph4num3r1c4l_1s_n0t_4_pr0bl3m}
```

## 方法总结

扩展字符集没有改变维吉尼亚的代数结构，只把模数从 26 改成字符表长度。必须使用源码给定的精确顺序，否则同一字符会映射到错误索引。
