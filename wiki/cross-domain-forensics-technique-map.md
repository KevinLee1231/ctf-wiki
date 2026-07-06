---
type: family
tags: [forensics, family, map]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/cross-domain-forensics-technique-map.md
  - ../raw/forensics/SU_forensicsWP.md
  - ../raw/forensics/SU_LightNovelWP.md
updated: 2026-07-06
---

# Cross-Domain Technique Map

## 作用边界

题目跨越多个取证介质，单一页面不足以覆盖：例如 PCAP 里传磁盘镜像、内存 dump 中藏浏览器历史、视频里出现硬件信号、容器层里残留凭据。这个页面应作为“下一跳地图”，帮助快速选择更具体技巧，而不是承载完整解法。

## 识别信号

- 附件类型多，或者一个文件明显包含另一个领域的证据。
- 首轮 `file` / `binwalk` / `strings` 只说明“有嵌套”，但还不能确定最终路径。
- 多个候选方向都能解释一部分现象：网络、磁盘、内存、图像、音频、硬件信号交错出现。
- raw 中的技巧名更像索引：Docker、RAID、GIMP raw dump、I2C、USB HID、DNS trailing byte、TLS master key。

## 最小证据

- 至少完成载体拆分：知道当前是在 PCAP、镜像、内存、媒体、日志还是容器层。
- 对每一层保存一个可复现中间产物，例如导出的 TCP stream、mount 后目录树、frame dump、音频频谱图。
- 每次 pivot 都要能说明“为什么从 A 跳到 B”，例如协议里出现文件头、内存里出现 AES key、视频里出现 LED Morse。

## 分流流程

1. 先做低成本分层：`file`、`binwalk`、`7z l`、`tshark -r`、`strings`、`exiftool`。
2. 为每一层建立 evidence log：输入文件、导出命令、输出路径、观察到的 flag-like/secret-like 线索。
3. 选最强证据进入专项页：PCAP 走协议重组，磁盘走 Sleuth Kit，内存走 Volatility/raw carving，媒体走图像/音频/视频隐写。
4. 如果工具失败，改用原始字节/格式结构验证，不要被单个工具结论卡住。
5. 解出中间 secret 后再回到上一层验证它是否是密码、key、seed 或 flag。

## 跨域路线分流

| 跨域证据 | 下一跳判断 |
|---|---|
| 容器/镜像转文件系统 | Docker layer、SquashFS、disk image 先拆层，再看删除文件和历史命令。 |
| 网络转文件 | 从 PCAP 导出 HTTP/SMB/TCP stream，按 magic bytes 重组。 |
| 内存转密钥 | Volatility 失败时，直接在 dump 中找 key、session id、framebuffer 或脚本源码。 |
| 媒体转信号 | 视频/音频可能是 LED Morse、UART、SSTV、频率音符或打印轨迹。 |

## 常见陷阱

- 把跨域索引页当最终解法页，导致只浏览不验证。
- 每次换方向都丢失中间产物，无法复现。
- 工具扫描没有结果就结束，而不是回到格式结构和字节证据。
- 忽略“解出来的字符串可能只是下一层密码”。

## 关联技巧

- [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)
- [3d-printing.md](3d-printing.md)
- [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md)
- [blockchain-and-transaction-forensics.md](blockchain-and-transaction-forensics.md)
- [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)
- [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)
- [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)
- [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md)
- [video-document-and-media-stego.md](video-document-and-media-stego.md)
- [keyboard-mouse-audio-and-physical-puzzles.md](keyboard-mouse-audio-and-physical-puzzles.md)
- [signals-and-hardware.md](signals-and-hardware.md)
- [linux-git-browser-and-container-forensics.md](linux-git-browser-and-container-forensics.md)
- [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md)

## WP 案例沉淀

本节只抽取 raw WP 中可复用的识别信号和下一跳，不替代原始题解正文。

| Raw WP | 复用信号 | 下一跳 |
|---|---|---|
| [SU_forensicsWP](../raw/forensics/SU_forensicsWP.md) | AD1 Windows 系统盘综合取证，证据集中在事件日志、Notepad TabState、应用缓存、聊天数据库和 Ollama 痕迹。 | [windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md)、[disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md) |
| [SU_LightNovelWP](../raw/forensics/SU_LightNovelWP.md) | PCAP 中出现 Kerberos/NTLM、MSRPC 与 TSCH 远程任务调度，先做 stream、凭据和会话密钥恢复，再解析计划任务与 PowerShell/AES 逻辑。 | [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)、[windows-registry-logs-and-credentials.md](windows-registry-logs-and-credentials.md) |

## 原始资料
- [cross-domain-forensics-technique-map.md](../raw/forensics/cross-domain-forensics-technique-map.md)
- [SU_forensicsWP.md](../raw/forensics/SU_forensicsWP.md)
- [SU_LightNovelWP.md](../raw/forensics/SU_LightNovelWP.md)
