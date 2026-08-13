# GreyCTF2024 CashHat The Ripper WP

## 题目简述

附件是带密码的 ZIP，题名提示使用 John the Ripper。这里不是破坏 ZIP 加密算法，而是从归档提取密码校验数据，再用常见口令字典恢复弱密码。

## 解题过程

先把 ZIP 的校验信息转换成 John 可识别的格式：

```bash
zip2john challenge.zip > zip.hash
john zip.hash --wordlist=/usr/share/wordlists/rockyou.txt
john zip.hash --show
```

字典命中密码：

```text
123mango
```

随后解压并读取文件：

```bash
unzip -P 123mango challenge.zip
cat flag.txt
```

仓库中的原始归档经该密码解密后，`flag.txt` 的实际内容为：

```text
flag{W34k_P4ssw0rds_St4Nd_n0_Ch4nc3}
```

这道题确实使用 `flag{...}`，不是 GreyCTF 常见的 `grey{...}`；不应为了统一格式自行改写。

## 方法总结

密码归档取证应保留“提取哈希、恢复口令、用口令读取原件”的完整证据链。最终 flag 以附件内解出的文本为准，尤其要警惕赛事 README、第三方题解或惯用前缀与实际文件不一致。
