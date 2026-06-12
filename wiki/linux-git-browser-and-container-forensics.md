---
type: technique
tags: [forensics, technique]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/linux-git-browser-and-container-forensics.md
updated: 2026-05-21
---

# Linux, Git, Browser and Container Forensics

## 适用场景

主要工作是从文件、镜像、内存、PCAP、日志、多媒体或物理信号中恢复证据。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 附件是 pcap、disk image、memory dump、office/pdf/image/audio/video、日志或容器层。
- 需要 carving、重组、解码、时间线、凭据恢复或隐写检测。
- flag 藏在工件或历史状态中。
- 题面或 raw 线索能落到这些关键词之一：Linux / Git / 浏览器取证速查技巧族：日志、历史、编码与缓存、Log Analysis、Linux Attack Chain Forensics、Docker Image Forensics (Pragyan 2026)、Browser Credential Decryption、Firefox Browser History (places.sqlite)、USB Audio Extraction from PCAP、TFTP Netascii Decoding。

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
| Linux / Git / 浏览器取证速查技巧族：日志、历史、编码与缓存 | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Log Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Linux Attack Chain Forensics | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Docker Image Forensics (Pragyan 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Browser Credential Decryption | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Firefox Browser History (places.sqlite) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| USB Audio Extraction from PCAP | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| TFTP Netascii Decoding | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| TLS Traffic Decryption via Weak RSA | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| ROT18 Decoding | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Common Encodings | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Git Directory Recovery (UTCTF 2024) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| KeePass Database Extraction and Cracking (H7CTF 2025) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Git Reflog and fsck for Squashed Commit Recovery (BearCatCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Browser Artifact Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Chrome/Chromium | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Firefox | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Corrupted Git Blob Repair via Byte Brute-Force (CSAW CTF 2015) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| VBA Macro Forensics - Excel Cell Data to ELF Binary (Sharif CTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Ethereum / Blockchain Transaction Tracing (Defenit CTF 2020) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Python In-Memory Source Recovery via pyrasite (Insomni'hack 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| 浏览器取证技巧族：缓存、历史与凭据定位 | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Browser Forensics | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

## 常见陷阱

- 只按关键词跳页，没有先构造最小证据。
- 照搬 raw 中的一次性 payload，没有检查当前题的边界条件。
- 忽略相邻技巧之间的 pivot，导致在错误方向上继续投入时间。

## 关联技巧

- [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)
- [3d-printing.md](3d-printing.md)
- [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md)
- [blockchain-and-transaction-forensics.md](blockchain-and-transaction-forensics.md)
- [cross-domain-forensics-technique-map.md](cross-domain-forensics-technique-map.md)
- [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)

## 原始资料

- [linux-git-browser-and-container-forensics.md](../raw/forensics/linux-git-browser-and-container-forensics.md)
