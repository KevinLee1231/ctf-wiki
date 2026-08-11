# 地球上最后的夜晚

## 题目简述

附件包含 `No Password.pdf` 和一个加密 7z。PDF 中通过 wbStego 隐藏了压缩包密码；解压后得到看似正常的 DOCX，而真正的 flag 位于 DOCX ZIP 容器中的自定义 XML 文件。

## 解题过程

先用 wbStego 对 `No Password.pdf` 执行解码。文件名已经提示“No Password”，因此密码输入框保持为空并继续。提取出的 `out.txt` 内容为：

```text
Zip Password: OmR#O12#b3b%s*IW
```

使用该密码解开 7z，得到：

```text
Last Evenings on Earth.docx
```

在 Word 或 WPS 中打开时，文档内容没有异常。DOCX 本质上是 ZIP 容器，将扩展名改为 `.zip` 或直接用解压软件打开，然后枚举内部文件：

```powershell
Get-ChildItem -Recurse
```

除标准的 `word/document.xml`、样式和关系文件外，还存在非标准文件：

```text
word/secret.xml
```

读取该 XML 即可得到：

```text
hgame{mkLbn8hP2g!p9ezPHqHuBu66SeDA13u1}
```

## 方法总结

- 核心链路：wbStego 无密码提取 PDF 隐藏数据、解密 7z、按 ZIP 容器检查 DOCX、自定义 XML 中取 flag。
- 识别信号：Office 文档表面完全正常，但题目暗示 XML 或额外部件时，应检查 OOXML 包内部而不只搜索可见正文。
- 复用要点：文件名提示可能直接对应工具参数；对 DOCX、XLSX、PPTX 等 OOXML 格式，应重点比较标准目录与额外文件。
