---
type: family
tags: [forensics, family, stego, pdf, png, gif, svg, text]
skills: [ctf-stego, ctf-forensics]
raw:
  - ../raw/stego/pdf-png-gif-and-text-stego.md
  - ../raw/forensics/HGAME2026-redacted-wp.md
updated: 2026-07-27
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
| PDF 有遮挡、metadata、link、object stream 或 post-EOF 数据 | 先用当前已确认的 `mutool`、`exiftool` 分层；若已安装再补 `pdfinfo`、`pdfimages`，随后检查 URI annotation、Flate stream、图片 LSB 和 redaction overlay。 | [forensics-tooling.md](forensics-tooling.md) |
| PNG/APNG chunk、CRC、高度、overlay、unknown chunk 异常 | 解析 chunk 顺序和 IHDR/IDAT/IEND，检查 APNG frame、EOF overlay、CRC/高度、custom chunk 和 XOR 层。 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) |
| GIF 帧、palette 或 frame count 异常 | 先拆帧和 palette，判断是 frame diff、palette-to-pixel、Morse、QR 还是 ELF/文件重组。 | [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md) |
| SVG/terminal/text 视觉正常但原始文本异常 | 直接看 XML/ANSI/Kitty escape 原始字节，排除渲染器隐藏、微坐标和不可见字符。 | [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md) |
| 视频容器看似无关 | 先 `ffprobe` 列所有 stream，尝试 `-map 0:N` 提取非默认视频/字幕/附件。 | [video-document-and-media-stego.md](video-document-and-media-stego.md) |
| spreadsheet/文本频率或多层交织 | 先统计唯一值、频率、行列/字节交织，再判断是否恢复二进制、图像或压缩包。 | [file-signatures-and-flag-artifact-hunting.md](file-signatures-and-flag-artifact-hunting.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [HGAME2026-redacted-wp](../raw/forensics/HGAME2026-redacted-wp.md) | PDF 脱敏题先区分视觉遮挡、文本对象残留、ToUnicode/CMap 字体反复制和增量保存历史版本；不要只跑 `strings` 或截图 OCR。 |
| [D3CTF2023-d3gif-wp](../raw/stego/D3CTF2023-d3gif-wp.md) | GIF/PNG/QR 媒体层叠，先拆帧和图像通道，再恢复二维码或文本 artifact。 |
| [0xGame2023-week1-重生之我在教学楼打-CS-wp](../raw/stego/0xGame2023-week1-重生之我在教学楼打-CS-wp.md) | 遇到陌生附件应先通过扩展名、文件头和题目语境识别载体，再选择原生程序打开。三维场景隐写的重点是系统搜索出生点、边界、物体底部和不可见面，并通过改变视角、亮度与对比度提高可读性。 |
| [UMDCTF2021-magic-wp](../raw/stego/UMDCTF2021-magic-wp.md) | 自立体图依靠空间视差，而不是最低位或元数据。先检查纹理是否具有稳定横向周期，再用差分/平移叠加寻找深度图。保留原始分辨率非常重要，缩放和重采样会破坏周期并降低隐藏轮廓的可见性。 |
| [UMDCTF2022-rsi-1-wp](../raw/stego/UMDCTF2022-rsi-1-wp.md) | 专有容器题应先按格式边界提取目标字段，不能对整个文件盲目解压。OSR 字符串使用 ULEB128 变长长度，若误当成单字节，后续偏移会在较长字段上失效。本题的 replay data 不含正常的逗号分隔帧，反而是一串空字节和明文，这也是它被人为用作隐藏载体的证据。 |
| [UMDCTF2022-rsi-2-wp](../raw/stego/UMDCTF2022-rsi-2-wp.md) | 与 RSI 1 相比，本题不能把解压结果直接当作文本，而要解释回放帧语义。绘图时应过滤哨兵、按键释放帧并在笔画间断线，否则跨字符连线会降低可读性。坐标系方向也是关键：若不反转 $y$ 轴，文字会垂直镜像。 |
| [UMDCTF2023-march-of-the-eight-wp](../raw/stego/UMDCTF2023-march-of-the-eight-wp.md) | 用文本提示确定“重音选择 → F 大调级数 → 三位八进制 ASCII”的完整视觉隐写通道；乐谱中大量刻意添加的重音符号、调性提示和异常大小写文字同时出现时，应把音乐理论元素视为数据选择与编码规则。 |
| [UMDCTF2026-closing-bell-wp](../raw/stego/UMDCTF2026-closing-bell-wp.md) | 本题分为三层：用校准字节拟合多 venue 仿射信道、穷举 24 位流密码种子恢复 ELF、再逆向 ELF 内可逆的 book 状态更新。每层都有独立校验：多字段评分、ELF 魔数与 checksum、目标哈希。由于最主要的工作是从伪装成行情的隐蔽通道重组载荷，最终归入 Stego；ELF 逆向是后续阶段。利用分层校验逐步收紧，比直接在 16 MB 文本中搜索可打印字符串可靠得多。 |
| [WMCTF2024-steg-allinone-wp](../raw/stego/WMCTF2024-steg-allinone-wp.md) | 第一层给参数，第二层给第三层参数，必须按顺序解；额外 IDAT 块不是普通图片显示信息，而是第三层所需原始蓝色通道。 |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| PDF/Office revision、未渲染对象或历史版本保留内容 | [structured-document-history-and-hidden-object-recovery.md](structured-document-history-and-hidden-object-recovery.md) |
| PNG/GIF/图像通道、bitplane 或帧差隐藏数据 | [media-channel-bitplane-and-frame-difference-extraction.md](media-channel-bitplane-and-frame-difference-extraction.md) |
| 文档/图片容器边界、尾随数据或内嵌对象决定提取 | [file-format-and-embedded-payload-identification.md](file-format-and-embedded-payload-identification.md) |

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

- [pdf-png-gif-and-text-stego.md](../raw/stego/pdf-png-gif-and-text-stego.md)
- [HGAME2026-redacted-wp.md](../raw/forensics/HGAME2026-redacted-wp.md)
