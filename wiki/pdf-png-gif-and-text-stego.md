---
type: family
tags: [forensics, family, stego, pdf, png, gif, svg, text]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/pdf-png-gif-and-text-stego.md
  - ../raw/misc/HGAME2026-redacted-wp.md
updated: 2026-07-06
---

# PDF, PNG, GIF and Text Stego

## 作用边界

本页是文档/图片/文本隐写 family，用于判断 PDF、PNG/APNG、GIF、SVG、终端转义文本、表格、视频容器和多层文件叠加中的隐藏信息路径。它承担格式族分流，不把每个 raw 标题当作一个独立 technique。

如果核心已经是像素位平面、JPEG DCT、QR 重建或图像拼图，优先转到 [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)。如果核心是音频频谱、SSTV、DTMF 或压缩包嵌套，转到对应媒体/文件页面。

## 识别信号

- 文件类型是 PDF、PNG/APNG、GIF、SVG、MP4/MKV、spreadsheet、terminal capture、ANSI art 或看似普通文本。
- `file` 识别正常但视觉内容太干净、帧数异常、chunk/object/stream 结构异常、metadata 异常、EOF 后有数据或容器内有多 stream。
- 题面提示 redaction、layer、frame、palette、animation、terminal、magic eye、QR、overlay、metadata、hidden in document/text。
- 单个工具扫不到时，格式结构仍有可解释异常：PDF object stream、PNG ancillary chunk、GIF palette、SVG 微坐标、ANSI escape、Kitty graphics、MP4 stream map。

## 最小证据

- 先确认真实格式和容器层：`file`、magic bytes、`exiftool`、`binwalk`、`ffprobe`、PDF object/chunk/frame 枚举。
- 保存每一层导出物：PDF 图片/stream、PNG chunk、GIF frame、SVG 放大视图、视频 stream、终端原始字节。
- 对“隐写”结论给出可复现差异：可见层与隐藏层的对象、坐标、bitplane、palette、frame diff、metadata 或 post-EOF 数据。
- 发现中间字符串后先判断它是 flag、密码、key、下一层文件名还是解密参数。

## 首轮路由

| 证据形态 | 先做什么 | 下一跳 |
|---|---|---|
| PDF 有遮挡、metadata、link、object stream 或 post-EOF 数据 | 先用 `pdfinfo/exiftool/pdfimages/mutool` 分层，再查 URI annotation、Flate stream、图片 LSB、redaction overlay。 | [forensics-tooling.md](forensics-tooling.md) |
| PNG/APNG chunk、CRC、高度、overlay、unknown chunk 异常 | 解析 chunk 顺序和 IHDR/IDAT/IEND，检查 APNG frame、EOF overlay、CRC/高度、custom chunk 和 XOR 层。 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) |
| GIF 帧、palette 或 frame count 异常 | 先拆帧和 palette，判断是 frame diff、palette-to-pixel、Morse、QR 还是 ELF/文件重组。 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) |
| SVG/terminal/text 视觉正常但原始文本异常 | 直接看 XML/ANSI/Kitty escape 原始字节，排除渲染器隐藏、微坐标和不可见字符。 | [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md) |
| 视频容器看似无关 | 先 `ffprobe` 列所有 stream，尝试 `-map 0:N` 提取非默认视频/字幕/附件。 | [video-document-and-media-stego.md](video-document-and-media-stego.md) |
| spreadsheet/文本频率或多层交织 | 先统计唯一值、频率、行列/字节交织，再判断是否恢复二进制、图像或压缩包。 | [file-signatures-and-flag-artifact-hunting.md](file-signatures-and-flag-artifact-hunting.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [HGAME2026-redacted-wp](../raw/misc/HGAME2026-redacted-wp.md) | PDF 脱敏题先区分视觉遮挡、文本对象残留、ToUnicode/CMap 字体反复制和增量保存历史版本；不要只跑 `strings` 或截图 OCR。 |
| [D3CTF2023-d3gif-wp](../raw/misc/D3CTF2023-d3gif-wp.md) | GIF/PNG/QR 媒体层叠，先拆帧和图像通道，再恢复二维码或文本 artifact。 |

## 合并与拆分结论

- 保留为 `family`：raw 覆盖 PDF、PNG/APNG、GIF、SVG、terminal/text、spreadsheet、video stream 和 file overlay，第一步都是格式结构分层，但后续工具和证据形态不同。
- 不并入 [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)：该页更适合像素/位平面/JPEG/QR 具体图像恢复；本页负责文档和容器格式的二级分流。
- 不拆成 PDF/PNG/GIF 三个页面：当前 raw 更像多格式速查族，且很多题会从一种容器跳到另一种隐藏层。

## 常见陷阱

- 只运行 `strings/binwalk`，没有解析 PDF object、PNG chunk、GIF frame 或 MP4 stream map。
- PDF redaction 只看截图，不检查 overlay、annotation、compressed stream 和 metadata。
- PNG 修 CRC 或高度后没有重新导出/查看完整画布，误以为修复失败。
- GIF 只看第一帧；palette、帧差、帧数平方、帧顺序都可能承载数据。
- 终端艺术复制到普通文本后丢失 escape sequence，破坏原始证据。

## 关联技巧

- [cross-domain-forensics-technique-map.md](cross-domain-forensics-technique-map.md)
- [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)
- [video-document-and-media-stego.md](video-document-and-media-stego.md)
- [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md)
- [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)
- [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md)
- [forensics-tooling.md](forensics-tooling.md)

## 原始资料

- [pdf-png-gif-and-text-stego.md](../raw/forensics/pdf-png-gif-and-text-stego.md)
- [HGAME2026-redacted-wp.md](../raw/misc/HGAME2026-redacted-wp.md)
