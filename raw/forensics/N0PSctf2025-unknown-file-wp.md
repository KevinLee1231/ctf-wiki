# N0PSctf2025 Unknown file Writeup

## 题目简述

题目给出一个没有扩展名的文件，要求判断其真实格式并读取内容。决定性线索不在文件名，而在文件头魔数。

## 解题过程

先查看文件开头：

```powershell
Format-Hex -LiteralPath ./challenge -Count 32
```

可以看到前八个字节为：

```text
25 50 44 46 2D 31 2E 36    %PDF-1.6
```

`%PDF-1.6` 是 PDF 文档的标准文件头。因此只需把文件重命名为 `challenge.pdf`，或直接让 PDF 阅读器按内容格式打开：

```powershell
Copy-Item -LiteralPath ./challenge -Destination ./challenge.pdf
```

PDF 只有一页，页面文本为：

```text
Congratulations!
The flag is B4BY{h1dD3n_PDF!}
```

因此 flag 为：

```text
B4BY{h1dD3n_PDF!}
```

仓库内官方 Markdown 末尾误粘贴了另一道 Caesar 密码题的 flag；这里以实际附件的 PDF 文本层和页面视觉结果为准。

## 方法总结

- 核心技巧：依据文件魔数识别无扩展名或错误扩展名的文件。
- 识别信号：文件名缺失、系统无法直接打开，但文件头包含 `%PDF-`、`PK`、PNG 或 ELF 等稳定签名。
- 复用要点：扩展名只影响默认打开方式，不决定真实格式；结论应由文件头和实际解析结果共同验证。
