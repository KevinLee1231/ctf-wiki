# Vinegar

## 题目简述

题名谐音提示 Vigenère Cipher。附件给出小写密文 `nmivrxbiaatjvvbcjsf` 和密钥 `secretkey`，需要按标准维吉尼亚规则解密。

## 解题过程

把字母映射为 $0$ 到 $25$，重复密钥并计算：

$$p_i=(c_i-k_i)\bmod26.$$

```python
ciphertext = "nmivrxbiaatjvvbcjsf"
key = "secretkey"
plain = "".join(
    chr((ord(c) - 97 - (ord(key[i % len(key)]) - 97)) % 26 + 97)
    for i, c in enumerate(ciphertext)
)
print(plain)
```

明文为 `vigenerecipherisfun`，按题面补上外壳：

```text
n00bz{vigenerecipherisfun}
```

## 方法总结

题目已经给出密钥，因此关键是识别古典密码并保持密钥循环对齐。解密使用减法而非加法，所有下标都要在 26 上取模。
