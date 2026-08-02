# nothing-to-see

## 题目简述

图片本身只显示 `nothing to see here`。生成脚本先清除原元数据，把带密码的 ZIP 直接追加到 PNG 文件尾部，再把 ZIP 密码写入 PNG 的 Title 元数据。决定性障碍是定位并提取被故意藏在图像容器后的归档载荷。

## 解题过程

先检查元数据和文件尾：

```bash
exiftool nothing.png
binwalk nothing.png
```

Title 字段给出：

```text
panda_d02b3ab3
```

PNG 解码器读到 `IEND` 后会忽略尾随数据，但 ZIP 工具可以直接从文件末尾识别中央目录。可用 `binwalk -e` 提取，也可以按 `PK\x03\x04` 偏移切出归档，然后解密：

```bash
binwalk -e nothing.png
unzip -P panda_d02b3ab3 hidden.zip
```

`flag.txt` 内容为：

```text
tjctf{the_end_is_not_the_end_4c261b91}
```

图片只承担纯文字掩护，视觉内容可在正文中完整表达，因此不保留重复截图。

## 方法总结

- 正常显示的媒体文件仍可能在规范结束标记后附加其他格式；magic、文件尾和 `binwalk` 是低成本首检项。
- 元数据既可能是载荷，也可能是下一阶段口令；提取归档前应一并查看 EXIF/XMP 字段。
- 识别到 ZIP 后仍要验证口令来源和解压结果，不应只凭文件扩展名判断成功。
