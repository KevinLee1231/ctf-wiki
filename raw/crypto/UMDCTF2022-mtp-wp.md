# Multiple Time Pad (MTP)

## 题目简述

题目用同一条 30 字节随机密钥分别异或八句长度相同的英文：

$$
C_i[j]=P_i[j]\oplus K[j].
$$

一次性密码本被重复使用后，两条密文异或会消去密钥。恢复八句明文后，需要按顺序直接拼接并计算 MD5，摘要才是提交内容。

## 解题过程

任取两条密文：

$$
C_a[j]\oplus C_b[j]=P_a[j]\oplus P_b[j].
$$

英文空格为 `0x20`，它与字母异或会翻转 ASCII 大小写位。因此在同一列中，若某条密文与多条其他密文异或后频繁落入字母范围，就很可能说明该条明文在该位置是空格。用多数投票先恢复空格，再把推导出的字母传播到其他句子。

剩余空位使用 crib dragging：根据已经显出的词形、标点和英文语法猜一个高置信字符，再利用

$$
P_b[j]=P_a[j]\oplus C_a[j]\oplus C_b[j]
$$

同步填充整列。最终八句为：

```text
Chungus is the god of thunder.
Earl grey tea is good for him.
March is a cold season for me.
Go and watch boba fett please.
I am someone who likes to eat!
Professor Katz taught me this.
All I got on the exam was a B.
Cryptography is a cool course!
```

严格按上述顺序无分隔符拼接，计算 MD5：

```python
import hashlib

plaintexts = [
    "Chungus is the god of thunder.",
    "Earl grey tea is good for him.",
    "March is a cold season for me.",
    "Go and watch boba fett please.",
    "I am someone who likes to eat!",
    "Professor Katz taught me this.",
    "All I got on the exam was a B.",
    "Cryptography is a cool course!",
]

digest = hashlib.md5("".join(plaintexts).encode()).hexdigest()
print(f"UMDCTF{{{digest}}}")
```

输出：

```text
UMDCTF{0a46e0b2b19dc21b5c15435653ffed67}
```

## 方法总结

- 重复使用 OTP/流密码密钥时，密文两两异或会消去密钥，转化为多明文联合恢复问题。
- 空格与字母异或的 ASCII 特征适合先做统计投票，再用词形和语法进行 crib dragging。
- 每确认一个位置的任意明文字节，就能立即恢复该列的密钥字节并传播到全部密文。
- 最终摘要对大小写、标点、顺序和是否插入换行都敏感，必须按题目给定的拼接规则计算。
