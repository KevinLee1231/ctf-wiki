# 所见即为假

## 题目简述

附件是一份伪加密 ZIP。修复后得到 `FLAG_IN_PICTURE.jpg`，压缩包注释同时给出 F5 隐写密钥。F5 提取结果又是一段十六进制编码的 RAR 文件，最后从归档内的文本取得 flag。

## 解题过程

部分解压软件会把 ZIP 的 general-purpose bit flag 误判为“已加密”。7-Zip 可以忽略该异常直接解压；也可以在十六进制编辑器中定位中央目录头：

```text
50 4B 01 02 3F 00 14 00 09 00
```

把末尾标志位 `09 00` 改为 `00 00`，或使用 WinRAR 的“修复压缩文件”功能，即可恢复正常 ZIP。压缩包注释给出的 F5 key 为：

```text
NllD7CQon6dBsFLr
```

使用 F5-steganography 的提取器：

```bash
java Extract -p NllD7CQon6dBsFLr -e out.txt FLAG_IN_PICTURE.jpg
```

`out.txt` 全部由十六进制字符组成，并以：

```text
52617221
```

开头。解码后前四字节是 `Rar!`，即 RAR 签名：

```python
from pathlib import Path

hex_text = Path("out.txt").read_text().strip()
Path("flag.rar").write_bytes(bytes.fromhex(hex_text))
```

打开 `flag.rar`，读取其中的 `flag.txt`：

```text
hgame{4087^z#msw34ERtnFUyqpKUk2dmLPW60}
```

## 方法总结

- 核心链路：修复 ZIP 伪加密、从注释取得 F5 密钥、提取 JPEG 隐写数据、十六进制还原 RAR。
- 识别信号：ZIP 声称加密但没有密码线索，且不同解压器表现不一致时，应比较本地文件头与中央目录的标志位。
- 复用要点：每一层输出都先检查文件签名；`52617221` 已足以识别 RAR，无需把十六进制文本当普通字符串继续解码。
