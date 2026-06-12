---
type: technique
tags: [forensics, technique]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/disk-memory-vm-and-container-forensics.md
updated: 2026-05-21
---

# Disk, Memory, VM and Container Forensics

## 适用场景

主要工作是从文件、镜像、内存、PCAP、日志、多媒体或物理信号中恢复证据。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 附件是 pcap、disk image、memory dump、office/pdf/image/audio/video、日志或容器层。
- 需要 carving、重组、解码、时间线、凭据恢复或隐写检测。
- flag 藏在工件或历史状态中。
- 题面或 raw 线索能落到这些关键词之一：Memory Forensics (Volatility 3)、Disk Image Analysis、VM Forensics (OVA/VMDK)、VMware Snapshot Forensics、GIMP Raw Memory Dump Visual Inspection (INShAck 2018)、Coredump Analysis、Windows KAPE Triage Analysis (UTCTF 2026)、PowerShell Ransomware Analysis。

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
| Memory Forensics (Volatility 3) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Disk Image Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| VM Forensics (OVA/VMDK) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| VMware Snapshot Forensics | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| GIMP Raw Memory Dump Visual Inspection (INShAck 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Coredump Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Windows KAPE Triage Analysis (UTCTF 2026) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| PowerShell Ransomware Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Android Forensics | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Container Forensics (Docker) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Cloud Storage Forensics (AWS S3 / GCP / Azure) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| BSON (Binary JSON) Format Reconstruction (IceCTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| TrueCrypt / VeraCrypt Volume Mounting (GreHack CTF 2016) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Volatility mftparser Offset-Based Deleted File Recovery (BSides Delhi 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Brotli Blob Detection via ASCII-Art Signature (ASIS Finals 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| corkami/pocs MD5 PDF Collision Generation (35C3 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| See Also | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Disk / VM / Memory Forensics | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

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
- [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)

## 原始资料

- [disk-memory-vm-and-container-forensics.md](../raw/forensics/disk-memory-vm-and-container-forensics.md)
