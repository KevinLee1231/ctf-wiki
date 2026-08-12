# DownUnderCTF 2022 doxme Writeup

## 题目简述

题目只给出一个没有扩展名的 `doxme` 文件，并提示 Office。目标是识别真实格式，再检查 Office 文档包中的嵌入资源。

文件实际上是 OOXML Word 文档。`.docx` 并非单一二进制格式，而是包含 XML、关系文件和媒体资源的 ZIP 容器，因此仅在 Word 中查看可见页面并不足以覆盖全部内容。

## 解题过程

先检查文件签名，可以看到 ZIP/OOXML 常见的 `PK` 头。为便于应用识别，可复制一份并补上 `.docx` 扩展名：

```powershell
Copy-Item -LiteralPath .\doxme -Destination .\doxme.docx
```

用 Word 打开后，页面中只能看到 flag 的前半段：

```text
DUCTF{WOrd_D0Cs_Ar
```

随后把文档当作 ZIP 包展开：

```powershell
Expand-Archive -LiteralPath .\doxme.docx -DestinationPath .\doxme-unpacked
Get-ChildItem -LiteralPath .\doxme-unpacked\word\media
```

`word/media` 中有两张透明背景图片。逐张铺在浅色背景上观察后，可以确认第一张是文档中已经显示的前半段，第二张保存了剩余内容：

```text
3_R34L1Y_W3ird}
```

按原顺序拼接，得到完整 flag：

```text
DUCTF{WOrd_D0Cs_Ar3_R34L1Y_W3ird}
```

这些图片只承载可准确转写的纯文本，没有额外布局或视觉证据价值，因此正文直接记录文字，不再保留截图。

## 方法总结

处理 OOXML 文档时，应同时检查“应用呈现层”和“容器结构层”。扩展名只影响程序关联，文件签名和内部目录才决定格式；`word/media`、文档关系和 XML 中都可能存在正文界面未完整呈现的内容。透明图片还应换用对比背景观察，避免把黑底预览误判为空白。
