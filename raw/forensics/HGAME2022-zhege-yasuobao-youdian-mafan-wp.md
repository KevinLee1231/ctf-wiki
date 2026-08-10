# 这个压缩包有点麻烦

## 题目简述

附件由多层 ZIP、密码提示、已知明文、图片尾部附加数据和 ZIP 伪加密组成。完整链条依次需要六位数字爆破、字典攻击、ZipCrypto 已知明文攻击、从 JPEG 后提取内嵌 ZIP，以及清除 ZIP 头中的伪加密标志。

## 解题过程

第一层 ZIP 注释为：

```text
Pure numeric passwords within 6 digits are not safe!
```

这限定了六位纯数字口令空间，可以用 `fcrackzip` 爆破：

```bash
fcrackzip -u -v -b -c 1 -l 6-6 challenge.zip
```

得到第一层密码：

```text
483279
```

解压后，`README.txt` 提示不要把所有密码写下来，而附件同时提供 `password-note.txt`，因此对第二层执行字典攻击。得到的口令包含反引号和末尾反斜杠，适合放在代码块中原样记录：

```text
&-`;qpCKliw2yTR\
```

第三层的提示是：

```text
If you don't like to spend time compressing files, just stores them.
```

已知的 `README.txt` 与加密 ZIP 中同名文件长度、CRC32 均相同，而且压缩方式是 Store。先把已知文件以“仅存储”方式放入明文 ZIP，再用 bkcrack 恢复 ZipCrypto 三个内部密钥：

```bash
zip -0 known.zip README.txt
bkcrack -C flag.zip -c README.txt -P known.zip -p README.txt
```

恢复出的密钥为：

```text
060fd5e1 d1f696b7 12655d8d
```

无需继续爆破原始长口令，直接用密钥解出目标图片：

```bash
bkcrack -C flag.zip -c flag.jpg \
  -k 060fd5e1 d1f696b7 12655d8d \
  -d flag.jpg
```

`flag.jpg` 的 JPEG 结束标志之后还有数据。使用 `binwalk -e`、`foremost`，或从尾部的 `PK` 签名手工切出内嵌 ZIP。最后一层并未真正加密，只设置了 ZIP General Purpose Bit Flag 的加密位。需要同时修改：

- 本地文件头 `50 4b 03 04` 后的通用标志字段；
- 中央目录头 `50 4b 01 02` 后的通用标志字段。

若字段原值为 `0x0009`，严格做法是清除最低的 encryption bit，变为 `0x0008`；部分解压工具也接受把该字段整体改为 `0x0000`。两处保持一致后即可正常解压，得到：

```text
hgame{W0w!_y0U_Kn0w_z1p_3ncrYpt!}
```

原 PDF 没有显示最终图片中的文字；密码、ZipCrypto 密钥和 flag 通过 [YuGao 的多层压缩包复现](https://sxyugao.top/p/d379320f) 核对，关键值与格式修改位置均已写入正文。

## 方法总结

多层归档题要把每层提示和文件结构对应起来：口令空间提示用于爆破，密码本用于字典攻击，相同 CRC 与 Store 模式用于已知明文攻击，JPEG 尾部的 `PK` 用于识别附加归档，最后再检查 ZIP 标志位是否只是伪加密。ZipCrypto 恢复的是内部密钥，不必知道原密码也能解出指定文件。
