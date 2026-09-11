---
type: tooling
tags: [forensics, tooling, tools, environment]
skills: [ctf-forensics]
---

# Forensics Tooling

本页维护文件、PCAP、磁盘、内存、日志与文档证据恢复的本机工具。隐藏载荷提取见 [stego-tooling.md](stego-tooling.md)，硬件采集与外设见 [hardware-embedded-tooling.md](hardware-embedded-tooling.md)。

## 平台与 Python 能力

文件分析、离线 PCAP、PDF、图片与表格默认使用 Windows `D:/文档/新建文件夹/venv/Scripts/python.exe`（3.14.3）。不要把已清理的通用 Python 包重新装入 WSL `ctf-tools`。

| Windows venv 包 | 当前版本 | 作用 |
|---|---|---|
| `scapy` | `2.7.0` | 离线 PCAP 读取、报文字段和重组脚本；当前无 libpcap provider，未建立实时抓包能力 |
| `PyMuPDF` / `pypdf` / `pdfplumber` | `1.27.2.3` / `6.10.2` / `0.11.9` | PDF 文本、页面、对象、附件及表格解析，按格式选用 |
| `Pillow` | `12.2.0` | 图像格式、尺寸、模式与像素检查 |
| `openpyxl` | `3.1.5` | XLSX 工作表、单元格与公式读取 |
| `lxml` / `beautifulsoup4` | `6.0.4` / `4.14.3` | XML/HTML 证据结构解析 |
| `zstandard` | `0.25.0` | Zstd 数据流解压；ZIP/tar/gzip 可先用标准库 |

当前通用环境没有 Volatility、oletools、pcodedmp 等专用内存或 Office 宏分析入口；它们不再列为可执行工具。PDF 与图像任务使用上述现存 Python 包，不调用已不存在的 mutool、ImageMagick 或旧 Conda 文档命令。

```pwsh
Get-FileHash -LiteralPath "D:/题目路径/evidence.bin" -Algorithm SHA256
& "D:/文档/新建文件夹/venv/Scripts/python.exe" -c 'from scapy.utils import PcapReader; from itertools import islice; r=PcapReader("D:/题目路径/capture.pcap"); [print(p.summary()) for p in islice(r,5)]; r.close()'
& "D:/文档/新建文件夹/venv/Scripts/python.exe" -c 'import pymupdf; d=pymupdf.open("D:/题目路径/document.pdf"); print(d.page_count, d.metadata); print(d.embfile_names()); d.close()'
& "D:/文档/新建文件夹/venv/Scripts/python.exe" "D:/题目路径/recover_evidence.py"
```

## 已有 WSL 系统工具

下列是 APT 包或现存系统入口。直接从 `pwsh` 调用，不套 Conda，不用 pip 改系统 Python：

| 工具与绝对入口 | APT 包版本 | 用途 |
|---|---|---|
| `/usr/bin/file`、`/usr/bin/strings`、`/usr/bin/xxd` | `file 1:5.47-4`、`binutils 2.47-2`；xxd 已有 | 文件头、字符串与字节定位 |
| `/usr/bin/binwalk` | `2.4.3+dfsg1-3` | 嵌入文件与固件签名识别 |
| `/usr/bin/foremost`、`/usr/bin/scalpel` | `1.5.7-12+b1` / `1.60+git20240110.6960eb2-2` | 按文件签名雕刻；输出目录与规则需指定 |
| `/usr/bin/testdisk`、`/usr/bin/photorec` | `testdisk 7.2-2` | 分区分析和文件恢复；交互写操作按授权进行 |
| `/usr/bin/mmls`、`fsstat`、`fls`、`istat`、`icat`、`blkcat`、`blkcalc` | `sleuthkit 4.14.0+dfsg-0kali1` | 分区、文件系统、inode 和数据块取证 |
| `/usr/sbin/debugfs`、`/usr/sbin/e2fsck` | `e2fsprogs 1.47.4-1+b1` | ext 文件系统检查；原证据优先只读 |
| `/usr/bin/tshark` | `4.6.6-1` | PCAP 协议解析和字段导出 |
| `/usr/bin/tcpdump`、`/usr/bin/pcapfix` | `4.99.6-2` / `1.1.7-3` | 离线报文查看与损坏 PCAP 修复输出 |
| `/usr/bin/scapy` | `python3-scapy 2.7.0+dfsg1-1` | APT 工具及其依赖；一般离线脚本优先 Windows |
| `/usr/bin/exiftool` | `libimage-exiftool-perl 13.55+dfsg-1` | 文件元数据和嵌入标记 |
| `/usr/bin/7z`、`/usr/bin/unsquashfs` | `7zip 26.02+dfsg-2` / `squashfs-tools 1:4.7.5-1` | 归档与 SquashFS 列表、提取 |
| `/usr/bin/tar`、`gzip`、`dd`；`/usr/sbin/losetup` | 系统入口存在 | Linux 容器/镜像辅助；不用写盘参数处理原证据 |

哈希口令恢复使用 [crypto-tooling.md](crypto-tooling.md) 的 John/hashcat 入口；音视频、条码、zsteg 和 steghide 的用途与调用由 Stego 工具页维护，不在这里重复清单。

## PCAP、磁盘与归档调用

`D:/题目路径` 对应 WSL `/mnt/d/题目路径`。先列信息再提取，输出写到独立文件：

```pwsh
wsl -d kali-linux --exec /usr/bin/tshark -r "/mnt/d/题目路径/capture.pcap" -c 20 -T fields -e frame.number -e ip.src -e ip.dst -e _ws.col.Protocol
wsl -d kali-linux --exec /usr/bin/mmls "/mnt/d/题目路径/disk.img"
wsl -d kali-linux --exec /usr/bin/fls -o 2048 "/mnt/d/题目路径/disk.img"
wsl -d kali-linux --exec /usr/bin/7z l "/mnt/d/题目路径/archive.7z"
wsl -d kali-linux --exec /usr/bin/exiftool "/mnt/d/题目路径/evidence.bin"
```

`-o 2048` 只是示例分区起始扇区，必须替换为 mmls 及实际扇区大小支持的值；不要把字节偏移直接当扇区。确定 inode 后，二进制数据在 Linux 内重定向，避免经 PowerShell 文本管道损坏：

```pwsh
wsl -d kali-linux --exec bash -c '/usr/bin/icat -o 2048 "/mnt/d/题目路径/disk.img" 42 > "/mnt/d/题目路径/recovered.bin"'
wsl -d kali-linux --exec /usr/sbin/debugfs -R "stats" "/mnt/d/题目路径/partition.img"
wsl -d kali-linux --exec /usr/sbin/e2fsck -n "/mnt/d/题目路径/partition-copy.img"
```

示例 inode `42` 也须替换。debugfs 不加 `-w`，e2fsck 使用 `-n`；需要修复时先明确副本和写入范围，不把修复原镜像作为首检。

## 失败处理与转向

- 文件类型与扩展名冲突：查 [file-signatures-and-flag-artifact-hunting.md](file-signatures-and-flag-artifact-hunting.md)，保留哈希、真实文件头与中间产物。
- 协议字段为空或重组失败：确认链路层、捕获完整性、过滤条件与会话方向；查 [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)。
- 文件系统读取失败：先核对分区偏移、扇区大小、加密与格式；查 [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)。
- 内存、VM 或容器工具缺失：先明确镜像格式、符号与目标版本，查 [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)，再评估 Windows 最小依赖。
- WSL 新增、升级或重装须先说明 Linux 必要性、安装方式、环境、依赖影响并取得明确同意；“恢复证据”不自动授权安装。
