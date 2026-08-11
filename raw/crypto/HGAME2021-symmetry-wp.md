# 对称之美

## 题目简述

程序生成一段 16 字节随机密钥，并循环使用它与一篇包含 flag 的英文长文本异或。一次性密码只有在密钥永不复用时才安全；本题让相同密钥覆盖多个 16 字节分组，等价于把许多明文放在同一个 pad 下，因而可以进行 Many-Time Pad 攻击。

## 解题过程

设第 $j$ 个 16 字节分组为 $C_j=P_j\oplus K$。任取两组异或，密钥会被抵消：

$$
C_i\oplus C_j=P_i\oplus P_j
$$

先把完整密文每 16 字节切成一行十六进制文本；末尾不足 16 字节的部分也必须保留：

```python
def split_for_mtp(cipher: bytes, width: int = 16) -> None:
    with open("cipher.ciphertexts", "w", encoding="ascii") as output:
        for offset in range(0, len(cipher), width):
            print(cipher[offset:offset + width].hex(), file=output)
```

英文文本中的空格尤其有用。若某一位置很可能是空格，则该字节与 `0x20` 异或便给出对应密钥字节；同一列的其他分组会随之出现可读字符。反复利用高频空格、常见单词、题目名 `Symmetry` 和 `hgame{` 格式做 crib dragging，可以逐列修正 16 字节密钥。MTP Interactive 会先自动给出部分明文，允许直接在界面中键入猜测并把修正传播到同一列；其输入格式、交互方式和导出功能已在 [MTP 项目说明](https://github.com/CameronLonsdale/MTP) 中概括，因此即使不打开链接，按上述步骤也能手工完成相同分析。

官方 PDF 只记录了方法，没有保留最终明文。同期复盘补齐了被隐藏在英文文章中的结果，并说明需要把错误的 `Symm try` 修正为 `Symmetry`，再校正另一列后得到：

```text
hgame{X0r_i5-a_uS3fU1+4nd$fUNny_C1pH3r}
```

该缺失结果可在 [MiaoTony 的 HGAME 2021 Week 1 复盘](https://miaotony.xyz/2021/02/07/CTF_2021HgameWeek1/) 中交叉核对；正文已包含其对解法有用的信息，链接仅作为来源留存。

## 方法总结

循环异或的核心弱点不是 XOR 本身，而是同一密钥字节在多个自然语言位置复用。已知周期后，应按周期转置密文，让每一列共享一个密钥字节，再结合空格统计、可打印字符比例和上下文猜词逐列恢复。自动工具只能生成候选，最终仍要用完整语句和 flag 格式验证歧义位置。
