# week1Crypto Sign in

## 题目简述

附件是一份两页的密码学入门 PDF，属于阅读型签到题。文档介绍 CTF 与 Crypto 方向的基础知识，并在末尾直接给出 flag。

## 解题过程

逐页阅读 PDF。正文把密码学粗分为两部分：古典密码主要依赖字母替换或顺序变换；现代密码包括 AES、DES 等分组密码，LFSR、LCG、RC4 等流密码，RSA、ECC、ElGamal 等公钥算法，以及哈希和密码协议。

文档还指出入门所需的三类基础：数论与后续线性代数、抽象代数知识，能够使用 Python 及其大整数和第三方库，以及阅读英文技术资料的能力。末页 `HERE IS WHAT YOU WANT` 下方直接写有：

```text
0xGame{Welcom_to_Cryptogrphy_World_!}
```

这里的 `Welcom` 和 `Cryptogrphy` 都是原文拼写，提交时不能自行改成标准英文。

## 方法总结

阅读型签到题的关键是完整检查附件，尤其是末页、页脚和显眼提示语。即使 flag 中存在拼写错误，也应逐字符按原文提交。
