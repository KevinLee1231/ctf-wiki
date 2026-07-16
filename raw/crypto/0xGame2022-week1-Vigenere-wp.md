# week1Vigenère

## 题目简述

附件是一段较长的英文 Vigenère 密文。文本长度足以进行重合指数和字母频率分析，flag 被嵌在解密后的英文段落中。

## 解题过程

只统计英文字母，对候选密钥长度分别计算各列的平均重合指数。长度为 8 时约为 $0.0686$，接近英文自然文本，而多数错误长度约为 $0.044$，因此密钥长度取 8。

把密文按下标模 8 分成八列，对每列枚举 26 个凯撒位移，并用英文频率做卡方评分，得到密钥：

```text
gmzzgnsc
```

随后按 $P_i=(C_i-K_{i\bmod 8})\bmod 26$ 解密。非字母字符原样保留，且不消耗密钥位置：

```python
ciphertext = open("Vigenère.txt", encoding="utf-8").read()
key = "gmzzgnsc"
plain = []
j = 0

for ch in ciphertext:
    if ch.isascii() and ch.isalpha():
        base = ord("A") if ch.isupper() else ord("a")
        shift = ord(key[j % len(key)]) - ord("a")
        plain.append(chr((ord(ch) - base - shift) % 26 + base))
        j += 1
    else:
        plain.append(ch)

text = "".join(plain)
print(text)
```

明文是介绍 CTF 的英文段落，其中出现：

```text
0xGame{V19eneRe_E4sy}
```

原题解使用在线求解器，但该网址并非复现解法所必需，且服务可用性不稳定，因此不在 WP 中保留；关键的密钥长度判断、密钥和解密规则已完整写入正文。

## 方法总结

- 核心方法：用重合指数判断密钥长度，再对各列做单表频率分析恢复 Vigenère 密钥。
- 识别特征：长密文只改变英文字母，保留空格和标点；按正确周期分列后，每列呈现接近英文的单表替换频率。
- 注意事项：要确认非字母是否推进密钥下标，并保留原始大小写；仅粘贴在线解密器链接无法独立复现，也不能说明为何结果可信。
