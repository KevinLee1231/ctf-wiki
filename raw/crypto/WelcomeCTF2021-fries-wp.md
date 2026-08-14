# Fries

## 题目简述

WelcomeCTF2021 的 Fries 给出一份经过单表代换的英文词表、同一套加密逻辑和加密后的 flag。词表末尾五个明文单词组成共享秘密，程序再以其 SHA-512 摘要作为异或密钥加密 flag。

## 解题过程

阅读 `fries.py` 可见，每个字母总被替换为 `alpha` 中同一位置的字符，所以词表使用的是固定单表代换，而不是逐字符变化的流密码。词界和重复模式均被保留，可以利用大量英文单词的词频、字母频率和单词形状恢复代换表。

官方解法使用简单替换密码求解器处理整份 `fries.txt`。这类求解器的关键工作是：根据单字母频率提出初始映射，再用词典匹配和语言评分迭代交换字母，直到密文单词整体能映射为高概率英文。求解完成后，无需依赖求解器继续解密，直接取明文词表最后五个单词并用空格连接：

```python
words = open("fries.txt", "r").read()
solver = SimpleSolver(words)
solver.solve()
shared_secret = " ".join(solver.plaintext().split()[-5:])
```

`encryptFlag` 实际是可逆的异或：

```python
digest = hashlib.sha512(shared_secret.encode("ascii")).digest()
ciphertext = open("encrypted_flag", "rb").read()
flag = bytes(c ^ k for c, k in zip(ciphertext, digest))
```

由于加密端和解密端对同一字符串计算同一摘要，异或一次即可恢复：

```text
greyhats{M@yb3_y0u_c@n_7rY_5paN15h}
```

## 方法总结

本题需要先识别单表代换，再沿源码追踪共享秘密的生成方式。大量词表不是噪声，而是为统计和词典约束提供语料；真正用于 flag 的只有恢复后最后五个单词。外部在线求解器并非必需，其重要能力已经体现在正文：用词频、单词模式和语言评分求出固定字母映射。
