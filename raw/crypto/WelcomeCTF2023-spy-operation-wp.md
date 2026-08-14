# Spy Operation

## 题目简述

附件包含一个使用随机密码加密的 ZIP，以及与压缩包内同名、内容完全相同的 `report.txt`。生成脚本使用 `pyminizip` 创建传统 ZipCrypto 加密包，随机密码本身无法猜测，但已知明文使 ZipCrypto 的内部三个 32 位密钥可以被恢复。

这道题的决定性障碍是传统 ZIP 加密的已知明文攻击，而不是通用文件取证，因此归入 Crypto。

## 解题过程

先用 `bkcrack` 查看条目和加密/压缩方式：

```bash
bkcrack -L secret.zip
```

`report.txt` 同时存在于压缩包外和压缩包内，可直接把外部文件作为完整已知明文。`bkcrack` 至少需要约 12 字节连续已知明文；完整报告远超这一要求。运行：

```bash
bkcrack -C secret.zip -c report.txt -p report.txt
```

命令会输出三项 ZipCrypto 内部密钥，例如：

```text
Keys: <key0> <key1> <key2>
```

恢复内部密钥后不必继续爆破原随机密码，可以直接生成无密码副本：

```bash
bkcrack -C secret.zip -k <key0> <key1> <key2> -D unlocked.zip
unzip unlocked.zip
```

解压后查看 `secret.png`，得到：

```text
greyhats{z1p_F1L3s_CHoS3n_pL41nT3xT_Att4ck}
```

以上命令与参数含义可在 [bkcrack 官方教程](https://github.com/kimci86/bkcrack/blob/master/example/tutorial.md) 中查证；关键点已经在正文给出，不需要依赖外链理解攻击流程。

## 方法总结

- 核心技巧：使用压缩包外的同名文件作为已知明文，恢复 ZipCrypto 内部密钥并直接解密归档。
- 识别信号：传统 ZipCrypto、压缩包内外存在相同文件、已知至少十余字节连续明文。
- 复用要点：内部密钥足以解密，不必恢复原密码；若条目使用 Deflate，还要确保已知明文与压缩后数据的对应方式满足工具要求。
