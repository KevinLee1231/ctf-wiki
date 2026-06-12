---
type: technique
tags: [forensics, technique]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/image-bitplane-qr-and-jpeg-stego.md
updated: 2026-05-21
---

# Image Bitplane, QR and JPEG Stego

## 适用场景

主要工作是从文件、镜像、内存、PCAP、日志、多媒体或物理信号中恢复证据。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 附件是 pcap、disk image、memory dump、office/pdf/image/audio/video、日志或容器层。
- 需要 carving、重组、解码、时间线、凭据恢复或隐写检测。
- flag 藏在工件或历史状态中。
- 题面或 raw 线索能落到这些关键词之一：JPEG Unused Quantization Table LSB Steganography (EHAX 2026)、BMP Bitplane QR Code Extraction + Steghide (BYPASS CTF 2025)、Image Jigsaw Puzzle Reassembly via Edge Matching (BYPASS CTF 2025)、F5 JPEG DCT Coefficient Ratio Detection (ApoorvCTF 2026)、PNG Unused Palette Entry Steganography (ApoorvCTF 2026)、QR Code Tile Reconstruction (UTCTF 2026)、Seed-Based Pixel Permutation + Multi-Bitplane QR (L3m0nCTF 2025)、JPEG Thumbnail Pixel-to-Text Mapping (RuCTF 2013)。

## 最小证据

- 已完成主方向判断，并确认本页技巧比相邻技巧更能解释当前证据。
- 至少有一个可复现输入、输出、文件结构、数学关系、协议行为或运行时状态。
- 能指出 raw 案例中哪一个变体与当前题最接近，以及不同点在哪里。

## 解法骨架

1. 识别格式和容器层。
2. 选择 carving、协议重组、内存插件或隐写分析。
3. 保留中间导出物和命令。
4. 把 recovered secret 再按需要解码或解密。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| JPEG Unused Quantization Table LSB Steganography (EHAX 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| BMP Bitplane QR Code Extraction + Steghide (BYPASS CTF 2025) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Image Jigsaw Puzzle Reassembly via Edge Matching (BYPASS CTF 2025) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| F5 JPEG DCT Coefficient Ratio Detection (ApoorvCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| PNG Unused Palette Entry Steganography (ApoorvCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| QR Code Tile Reconstruction (UTCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Seed-Based Pixel Permutation + Multi-Bitplane QR (L3m0nCTF 2025) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| JPEG Thumbnail Pixel-to-Text Mapping (RuCTF 2013) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Conditional LSB Extraction — Near-Black Pixel Filter (BaltCTF 2013) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| JPEG Slack Space Steganography (BSidesSF 2025) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Nearest-Neighbor Interpolation Steganography (BSidesSF 2025) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RGB Parity Steganography (Break In 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Pixel Coordinate Chain Steganography (H4ckIT CTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| AVI Frame Differential Pixel Steganography (H4ckIT CTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| JPEG Single-Bit-Flip Brute Force with OCR (SECCON 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| GIF Frame PLTE Chunk Concatenation to ELF (IceCTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Nested-Resize QR Overlay at Survivor Pixels (SECCON 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| ImageMagick +append Puzzle Stitching + gaps Solver (X-MAS CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Steghide Passphrase in JPEG Header Metadata (Saudi/Oman CTF 2019) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Corrupted PNG Magic and Lowercase Chunk Repair (Pragyan CTF 2019) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Steganography | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [3d-printing.md](3d-printing.md)
- [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md)
- [blockchain-and-transaction-forensics.md](blockchain-and-transaction-forensics.md)
- [cross-domain-forensics-technique-map.md](cross-domain-forensics-technique-map.md)
- [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)

## 原始资料

- [image-bitplane-qr-and-jpeg-stego.md](../raw/forensics/image-bitplane-qr-and-jpeg-stego.md)
