---
type: technique
tags: [forensics, technique]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/pdf-png-gif-and-text-stego.md
updated: 2026-05-21
---

# PDF, PNG, GIF and Text Stego

## 适用场景

主要工作是从文件、镜像、内存、PCAP、日志、多媒体或物理信号中恢复证据。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 附件是 pcap、disk image、memory dump、office/pdf/image/audio/video、日志或容器层。
- 需要 carving、重组、解码、时间线、凭据恢复或隐写检测。
- flag 藏在工件或历史状态中。
- 题面或 raw 线索能落到这些关键词之一：Quick Tools、Binary Border Steganography、Multi-Layer PDF Steganography (Pragyan 2026)、Advanced PDF Steganography (Nullcon 2026 rdctd series)、SVG Animation Keyframe Steganography (UTCTF 2024)、APNG (Animated PNG) Frame Extraction (IceCTF 2016)、PNG Height/CRC Manipulation for Hidden Content (H4ckIT CTF 2016)、PNG Chunk Reordering (0xFun 2026)。

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
| Quick Tools | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Binary Border Steganography | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Multi-Layer PDF Steganography (Pragyan 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Advanced PDF Steganography (Nullcon 2026 rdctd series) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| SVG Animation Keyframe Steganography (UTCTF 2024) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| APNG (Animated PNG) Frame Extraction (IceCTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| PNG Height/CRC Manipulation for Hidden Content (H4ckIT CTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| PNG Chunk Reordering (0xFun 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| File Format Overlays (0xFun 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Nested PNG with Iterating XOR Keys (VuwCTF 2025) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| GIF Frame Differential + Morse Code (BaltCTF 2013) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| GZSteg + Spammimic Text Steganography (VolgaCTF 2013) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Spreadsheet Frequency Analysis Binary Recovery (Sharif CTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Kitty Terminal Graphics Protocol Decoding (BSidesSF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| ANSI Escape Sequence Steganography in Terminal Art (BSidesSF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Autostereogram / Magic Eye Solving (BSidesSF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Two-Layer Byte+Line Interleaving (BSidesSF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Multi-Stream Video Container Steganography (BSidesSF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Progressive PNG Layered XOR Decryption (OpenCTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| QR Code Reconstruction from Curved Glass Reflection in Video (PlaidCTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| GIF Palette Manipulation for QR Code Reconstruction (3DSCTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Angecryption: AES-CBC Encrypting One Valid File into Another (34C3 CTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| SVG Micro-Coordinate Steganography (SharifCTF 8) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| PDF Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

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

- [pdf-png-gif-and-text-stego.md](../raw/forensics/pdf-png-gif-and-text-stego.md)
