# Freddie's Files

## 题目简述

题目给出一个受密码保护的 ZIP，以及一份与 Queen 乐队相关的文本。压缩包内仍有密文和 MD5 摘要，完整链路包括 ZIP 口令恢复、定制字典破解和大整数异或。

## 解题过程

先从 ZIP 提取 John 格式的哈希并破解：

```bash
zip2john FreddieFiles.zip > zip.hash
john zip.hash
```

得到压缩包密码：

```text
qfreddie78
```

解压后可见 `ciphertext.txt`、`key.txt` 和 `queen.txt`。`key.txt` 中的值：

```text
2796e7e1e7a2f60f4bab9bce1210c1a5
```

是 MD5 摘要。用题目提供的 `queen.txt` 作为定制字典破解，原文为：

```text
Buddy, you're a boy, make a big noise
```

`ciphertext.txt` 是一个二进制整数。这里不是把密钥循环到等长后逐字节异或，而是将密文位串和整段歌词都看作大端大整数，执行一次整体异或：

```python
c = int(cipher_bits, 2)
k = int.from_bytes(key_phrase.encode(), "big")
m = c ^ k
plain = m.to_bytes((m.bit_length() + 7) // 8, "big")
print(plain.decode())
```

输出：

```text
UMDCTF-{pHR3dd13_m3RcURY_w0uld_83_pR0uD}
```

## 方法总结

本题每个附件都对应一层线索：ZIP 哈希负责外层口令，主题文本负责 MD5 字典，恢复出的句子才是异或密钥。处理位串时必须先确认出题脚本的数值语义；整体整数异或与重复密钥异或的对齐方式不同，错误地从高位循环密钥不会得到正确明文。
