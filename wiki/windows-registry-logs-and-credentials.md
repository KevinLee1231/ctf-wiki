---
type: technique
tags: [forensics, technique]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/windows-registry-logs-and-credentials.md
updated: 2026-05-21
---

# Windows Registry, Logs and Credentials

## 适用场景

主要工作是从文件、镜像、内存、PCAP、日志、多媒体或物理信号中恢复证据。

本页不是 raw 的目录页；它把原始资料中的案例压缩成可迁移的判断信号、最小证据和解题骨架。

## 识别信号

- 附件是 pcap、disk image、memory dump、office/pdf/image/audio/video、日志或容器层。
- 需要 carving、重组、解码、时间线、凭据恢复或隐写检测。
- flag 藏在工件或历史状态中。
- 题面或 raw 线索能落到这些关键词之一：Windows Event Logs (.evtx)、Registry Analysis、OEMInformation Backdoor Detection、SAM Database Analysis、Recycle Bin Forensics、Browser History、Windows Telemetry (imprbeacons.dat)、Hosts File Hidden Data。

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
| Windows Event Logs (.evtx) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Registry Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| OEMInformation Backdoor Detection | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| SAM Database Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Recycle Bin Forensics | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Browser History | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Windows Telemetry (imprbeacons.dat) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Hosts File Hidden Data | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Contact Files (.contact) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| WinZip AES Encrypted Archives | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| NTFS Alternate Data Streams | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| NTFS MFT Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| USN Journal ($J) Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| SAM Account Creation Timing | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Impacket wmiexec.py Artifacts | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| PowerShell History as Timeline | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| User Profile Creation as First Login Indicator | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| RDP Session Event IDs | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Windows Defender MPLog Analysis | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Anti-Forensics Detection Checklist | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Windows Memory Forensics: certutil Base64 ZIP Recovery (SEC-T CTF 2017) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| NTFS EFSTMPWP Folder as cipher.exe Wipe Artifact (Security Fest CTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Volatility clipboard Plugin for Copy-Paste Secret Recovery (OtterCTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |
| Volatility Credential Recovery Toolkit (OtterCTF 2018) | 关注触发条件、最小 payload / 最小样本、失败信号和可自动化验证方式。 |

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

- [windows-registry-logs-and-credentials.md](../raw/forensics/windows-registry-logs-and-credentials.md)
