# Diceware

## 题目简述

题目给出一组看似普通的 Diceware 密码：

```text
trinity quote void excursion trustable chummy
```

提示指向 EFF，说明这些单词应当在 EFF Dice-Generated Passphrases 的大词表中反查。这里的关键不是把六个单词当作口令，而是利用每个单词对应的五位骰子编号继续还原数据。

## 解题过程

在 [EFF Large Wordlist](https://www.eff.org/files/2016/07/18/eff_large_wordlist.txt) 中逐个查找，得到：

```text
trinity    62616
quote      46265
void       65666
excursion  26152
trustable  62655
chummy     15565
```

把编号依次拼接：

```text
626164626565666261526265515565
```

这串字符全部落在十六进制数字范围内。每两个字符作为一个字节解码：

```python
codes = ["62616", "46265", "65666", "26152", "62655", "15565"]
secret = bytes.fromhex("".join(codes)).decode()
print(secret)
```

输出为：

```text
badbeefbaRbeQUe
```

因此 flag 为：

```text
UMDCTF-{badbeefbaRbeQUe}
```

计算其 SHA-256，可得到 README 中的 `141e93953a31274ee83e5f8b43e6467d4f19c64ed2cd847f504a52b17a5ebc49`。

## 方法总结

Diceware 单词表同时保存了“骰子编号到单词”的映射。题目利用单词隐藏一串只含 `1` 到 `6` 的十六进制文本；反查编号并拼接后，还要再识别一次十六进制编码。遇到类似题目时，不应停在“找到了单词编号”，还要检查编号串是否具有 ASCII、十六进制或其他结构。
