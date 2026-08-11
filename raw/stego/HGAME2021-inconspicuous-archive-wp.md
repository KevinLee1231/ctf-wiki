# 不起眼压缩包的养成的方法

## 题目简述

附件表面是一张 JPEG，实际在文件尾部附加了 ZIP。后续压缩包依次使用数字口令、ZipCrypto 已知明文攻击、ZIP 伪加密和 HTML 实体编码保护 flag。题目给出的每条注释都在提示下一层所需的攻击方式。

## 解题过程

先查看图片元数据，可以读到注释 `Secret hidden IN picture.`。使用 `binwalk`、`foremost` 或直接搜索 ZIP 文件头 `PK\x03\x04`，即可从图片尾部提取压缩包。压缩包注释为：

```text
Password is picture ID (Up to 8 digits)
```

“picture ID”指向图片来源编号；即使不知道来源，也可以把口令空间限制为最多 8 位十进制数。爆破得到第一层密码：

```text
70415155
```

解压后得到 `plain.zip` 和 `NO PASSWORD.txt`。`plain.zip` 内也有一个内容与 CRC32 均相同的 `NO PASSWORD.txt`，而提示 `By the way, I only use storage.` 说明构造已知明文压缩包时必须选择“仅存储”，使压缩方法与目标条目一致。用已知的明文文件对 ZipCrypto 做已知明文攻击，恢复第二层密码：

```text
C8uvP$DP
```

打开 `plain.zip` 后得到 `flag.zip`。另一条提示 `Because it's too strong or null. XD` 中，`strong` 对应上一层真实加密，`null` 则提示这一层只有 ZIP 伪加密标志。ZIP 的通用标志位在本地文件头 `50 4B 03 04` 和中央目录头 `50 4B 01 02` 中各保存一份，需同时清除 bit 0：本附件中本地头的 `09 00` 应改为 `08 00`，中央目录的 `01 00` 应改为 `00 00`。这样既移除“已加密”标记，又保留本地头中的数据描述符位。

解压出的 `flag.txt` 是 HTML 十六进制实体，可直接还原：

```python
import html

encoded = "<flag.txt 中的实体字符串>"
print(html.unescape(encoded))
```

得到：

```text
hgame{2IP_is_Usefu1_and_Me9umi_i5_W0r1d}
```

## 方法总结

嵌套压缩题要把每层提示与文件结构对应起来：图片尾部数据用文件签名提取，有限数字空间适合字典或爆破，重复文件可用于 ZipCrypto 已知明文攻击，而“能看到目录但解压索要密码”常见于伪加密。修改 ZIP 标志时应清除加密位而非盲目覆盖全部标志，以免同时破坏数据描述符等其他语义。
