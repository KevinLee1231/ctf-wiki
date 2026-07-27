---
type: technique
tags: [forensics, pdf, document, incremental-update, hidden-object, technique]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/HGAME2026-redacted-wp.md
  - ../raw/forensics/UMDCTF2017-the-lost-flag-wp.md
  - ../raw/forensics/UMDCTF2020-sensitive-wp.md
  - ../raw/forensics/UMDCTF2018-snoop-wp.md
updated: 2026-07-27
---

# Structured Document History and Hidden-Object Recovery

## 适用场景

PDF、动画或自定义结构化文档能正常打开，但可见页面只是当前引用图的一种视图。历史增量更新、孤立对象、被遮挡文本、损坏索引、字体映射、低对比度图层或非首帧内容仍可能保留关键证据。

## 识别信号

- PDF 存在多个 `startxref`、增量保存、对象流、嵌入字体或 redaction/遮挡。
- `pdftotext`、渲染页面和对象级提取结果互相矛盾。
- 文件尾、未引用对象或旧 xref 中存在 Flate/ASCII85 等编码流。
- 文档修复后可打开但视觉上仍有低对比度、二维码、隐藏层或后续帧。
- 自定义格式有明确 section、offset、帧或动画记录。

## 最小证据

- 保存原文件哈希，并区分“解析器当前视图”和“文件中实际存在的对象/字节”。
- 列出 xref、trailer、对象引用和增量更新链，或自定义格式的 section 边界。
- 对每个候选流记录对象号、过滤器、解码顺序和导出结果。
- 修复或提取后进行页面/逐帧视觉核对，不以“工具无报错”作为完成标准。

## 解法骨架

1. 先用多种解析视图比较：文本提取、页面渲染、对象列表、文件尾和历史版本。
2. 对 PDF 沿 `startxref`/trailer 回溯增量更新，扫描当前 xref 未引用的孤立对象。
3. 按对象声明的 filter 顺序解码 stream，必要时检查 ToUnicode CMap 与嵌入字体。
4. 对损坏或自定义格式按字段宽度、端序和 section 边界精确修复，确认最终 offset 等于文件长度。
5. 对恢复文档逐页、逐层、逐帧检查，并针对低对比度/二维码做增强和解码。

## 关键变体

- 视觉遮挡不等于内容删除，文本对象可能仍可选择或保留在内容流。
- 字体 glyph 与 Unicode 映射可被篡改，页面形状和复制文本会产生不同结果。
- 增量保存只追加新对象和 xref，旧版本内容通常仍在原字节中。
- 未激活动画帧或自定义 section 不会出现在默认查看器首屏。

## 常见陷阱

- 只运行 `strings` 或 `pdftotext`，没有查看对象图和历史 xref。
- 把可见黑框当作真正 redaction，不检查底层文本或图层。
- 解码 stream 时忽略 filter 顺序，得到看似乱码的中间层就停止。
- 修复文件能被解析就结束，没有做视觉和逐帧验收。
- 直接按值删除“损坏字节”，而不是按已知位置逆转破坏规则。

## 关联技巧

- [cross-domain-forensics-technique-map.md](cross-domain-forensics-technique-map.md)
- [file-signatures-and-flag-artifact-hunting.md](file-signatures-and-flag-artifact-hunting.md)
- [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)
- [video-document-and-media-stego.md](video-document-and-media-stego.md)
- [forensics-tooling.md](forensics-tooling.md)

## 原始资料

- [HGAME2026-redacted-wp.md](../raw/forensics/HGAME2026-redacted-wp.md)
- [UMDCTF2017-the-lost-flag-wp.md](../raw/forensics/UMDCTF2017-the-lost-flag-wp.md)
- [UMDCTF2020-sensitive-wp.md](../raw/forensics/UMDCTF2020-sensitive-wp.md)
- [UMDCTF2018-snoop-wp.md](../raw/forensics/UMDCTF2018-snoop-wp.md)
