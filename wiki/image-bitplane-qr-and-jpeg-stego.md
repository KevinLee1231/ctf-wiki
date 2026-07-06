---
type: family
tags: [forensics, family, image, stego, qr, jpeg, bitplane]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/image-bitplane-qr-and-jpeg-stego.md
  - ../raw/misc/HGAME2026-shiori不想找女友-wp.md
  - ../raw/misc/HGAME2026-invest-on-matrix-wp.md
  - ../raw/misc/VNCTF2026-mymnemonic-wp.md
updated: 2026-07-06
---

# Image Bitplane, QR and JPEG Stego

## 作用边界

本页是图像隐写和图像重组 family，覆盖 JPEG 量化表/DCT/slack/thumbnail、BMP/PNG bitplane、调色板、QR tile、像素置换、拼图重组、GIF/AVI 帧和 steghide 口令线索等路线。

它不替代 [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md) 的跨格式入口；本页负责进入图像后，判断该从像素、压缩结构、二维码结构、帧序列还是元数据口令继续。

## 识别信号

- 附件是 JPEG/PNG/BMP/GIF/AVI 或由图片碎片、二维码块、缩略图、调色板、DCT 系数、低位平面组成。
- 正常预览没有 flag，但元数据、通道、bitplane、DQT/PLTE、帧差、缩放残留或坐标链存在异常。
- 需要导出中间图像、重组 QR、提取 bitstream、修复 header/chunk 或再喂给 steghide/zbar/OCR。

## 最小证据

- 保留原始文件 hash 和未重压缩副本。
- 确认格式层：文件头、chunk/marker、EXIF、调色板、帧数、缩略图、尺寸和颜色模式。
- 对 bitplane/像素类，记录通道、bit 位、扫描顺序、阈值和端序。
- 对 QR/拼图类，记录网格尺寸、finder pattern、tile 顺序、旋转和 ECC/decoder 结果。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| JPEG DQT、DCT 系数、F5、slack、thumbnail | 先解析 marker 和未引用数据，再决定 bitstream、OCR 或 steghide | [forensics-tooling.md](forensics-tooling.md) |
| BMP/PNG LSB、bitplane、RGB parity、palette unused entry | 先按通道和 bit 位导出平面，检查是否形成文本、QR 或压缩包 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| QR tile、nested resize、multi-bitplane QR | 先恢复 finder/timing/网格，再用 zbar 或 known prefix 校验 | [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md) |
| 图像拼图、edge matching、坐标链 | 先建立边缘相似度或坐标遍历顺序，再导出重组图 | [interactive-containers-jails-and-solvers.md](interactive-containers-jails-and-solvers.md) |
| GIF/AVI 帧差、PLTE 拼接、视频帧藏文件 | 先导出帧和 palette/chunk，再判断是否转视频/容器页面 | [video-document-and-media-stego.md](video-document-and-media-stego.md) |
| PNG magic/chunk 损坏或大小写异常 | 先修 magic、critical chunk 和 CRC，再继续隐写分析 | [file-signatures-and-flag-artifact-hunting.md](file-signatures-and-flag-artifact-hunting.md) |

## 合并与拆分结论

本页应保留为 family。JPEG 结构、bitplane、QR 重组、帧差和图像拼图的第一步证据不同；但它们都属于图像证据源内的二级分流。当前不拆小页，避免把短案例拆成孤立节点。

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [HGAME2026-shiori不想找女友-wp](../raw/misc/HGAME2026-shiori不想找女友-wp.md) | EXIF 中的 JSON 参数直接描述像素抽样网格；先按 `start/step/column` 重排隐藏图像，再处理恢复出的线索。 |
| [HGAME2026-invest-on-matrix-wp](../raw/misc/HGAME2026-invest-on-matrix-wp.md) | 题面给出的矩阵不是代数约束，而是 QR 小块；按 `5x5` block 和编号顺序拼回二维码后再扫码。 |
| [VNCTF2026-mymnemonic-wp](../raw/misc/VNCTF2026-mymnemonic-wp.md) | 图片末尾黑白格按固定像素步长转 bitstream，图像阶段只负责稳定提取 192-bit ENT，后续 BIP39 checksum/seed 交给 crypto 长尾结构页。 |
| [D3CTF2022-badw3ter-wp](../raw/misc/D3CTF2022-badw3ter-wp.md) | 图片图层、黑底 QR 和隐写链组合，先分离图层/背景再恢复二维码或图像隐藏信息。 |
| [D3CTF2023-d3image-wp](../raw/misc/D3CTF2023-d3image-wp.md) | 图像/二维码/摩斯或像素模式是主线，先分离颜色层、定位图案再恢复可读编码。 |
| [D3CTF2025-d3image-wp](../raw/misc/D3CTF2025-d3image-wp.md) | 图像块变换和隐写编码可逆，先抽出块顺序/颜色关系再写正反向恢复脚本。 |

## 常见陷阱

- 用截图或编辑器另存图片，破坏原始低位和 chunk。
- 只跑 steghide/binwalk，没检查 bitplane、palette、thumbnail 和 DCT。
- QR 重组只按视觉猜，不利用 finder/timing 和 ECC 反馈。
- JPEG 题只看像素，忽略 marker、DQT 和压缩域。
- 解出 bitstream 后没继续按压缩包、Base64 或 XOR 检查。

## 关联技巧

- [cross-domain-forensics-technique-map.md](cross-domain-forensics-technique-map.md)
- [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md)
- [video-document-and-media-stego.md](video-document-and-media-stego.md)
- [exotic-encodings-and-file-formats.md](exotic-encodings-and-file-formats.md)
- [forensics-tooling.md](forensics-tooling.md)

## 原始资料

- [image-bitplane-qr-and-jpeg-stego.md](../raw/forensics/image-bitplane-qr-and-jpeg-stego.md)
- [HGAME2026-shiori不想找女友-wp](../raw/misc/HGAME2026-shiori不想找女友-wp.md)
- [HGAME2026-invest-on-matrix-wp](../raw/misc/HGAME2026-invest-on-matrix-wp.md)
- [VNCTF2026-mymnemonic-wp](../raw/misc/VNCTF2026-mymnemonic-wp.md)
