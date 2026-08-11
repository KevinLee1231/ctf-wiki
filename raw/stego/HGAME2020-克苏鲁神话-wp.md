# 克苏鲁神话

## 题目简述

附件包含明文 `Bacon.txt` 和加密的 `Novel.zip`；压缩包内部也有一个内容、大小均与外部文件相同的 `Bacon.txt`。这为传统 ZipCrypto 提供了已知明文。解密压缩包后，还要从大小写混排文本中解出培根密码，并在 Word 文档中显示隐藏文字。

## 解题过程

第一步是确认外部 `Bacon.txt` 与压缩包内同名条目的未加密内容一致。用相同压缩参数把已知明文制作成 `known.zip`，再对 ZipCrypto 做已知明文攻击。例如使用 `bkcrack`：

```bash
bkcrack -C Novel.zip -c Bacon.txt -P known.zip -p Bacon.txt
```

官方解题记录恢复出的三组内部密钥为：

```text
720b7516 d2d6a716 2c24dcae
```

可据此生成解密副本：

```bash
bkcrack -C Novel.zip -k 720b7516 d2d6a716 2c24dcae -D Novel-decrypted.zip
```

解压得到 `The Call of Cthulhu.doc`。文档仍受口令保护，而 `Bacon.txt` 中的提示文本大小写异常：

```text
of SuCh GrEAt powerS OR beiNGS tHere maY BE concEivAbly A SuRvIval oF HugeLy REmOTE periOd.

*Password in capital letters.
```

把小写映射为 `a`、大写映射为 `b`，忽略空格和标点，再每 5 位按培根密码表解码，得到全大写口令：

```text
FLAGHIDDENINDOC
```

用该口令打开 Word 文档后，正文仍看不到 flag。“HIDDEN”同时提示 Word 的隐藏文字属性；在文字处理软件的显示设置中启用“隐藏文字”，即可在文末看到：

```text
hgame{Y0u_h@Ve_F0Und_mY_S3cReT}
```

## 方法总结

- 核心技巧：用压缩包外的同内容文件攻击 ZipCrypto，再把大小写模式作为培根密码，最后显示 Word 隐藏文字。
- 识别信号：加密 ZIP 内外出现大小相同的同名文件；文本大小写刻意混排；解出的口令含有 `HIDDEN` 等操作提示。
- 复用要点：已知明文攻击依赖字节级一致性和正确的压缩条目；拿到文档口令不等于完成，还应检查隐藏文字、批注、修订和对象层。
