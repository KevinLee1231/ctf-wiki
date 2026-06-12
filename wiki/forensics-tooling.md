---
type: tooling
tags: [forensics, tooling, tools, environment]
skills: [ctf-forensics]
updated: 2026-05-21
---

# Forensics Tooling

本页记录 `ctf-forensics` 方向的本机工具清单、调用层、路径和适用边界。`SKILL.md` 只保留首轮工具摘要；需要详细路径、环境和专项工具说明时读取本页。

## 调用层与覆盖状态

### 非交互调用原则

- 首轮先用 system-global 工具识别载体：`file`、`exiftool`、`tshark`、`binwalk`、Sleuth Kit。
- 需要 Python 图像/信号/内存处理时再进入 `ctf-tools`，并显式激活/退出 Conda。
- 对磁盘镜像、内存 dump、PCAP 和媒体文件，优先导出中间产物，避免只保留工具 UI 结论。

### 知识页覆盖状态

- 当前覆盖磁盘、内存、PCAP、音视频、图像隐写、PDF、硬件信号、容器和文件系统。
- 后续缺口集中在 Windows 事件日志、云对象存储、移动端 app 数据库和更多现代文件格式。

### 后续补强方向

- Windows triage：EVTX、Amcache、ShimCache、PowerShell history。
- Cloud forensics：S3/GCS/Azure versioning 和公开 bucket 误配。
- Mobile forensics：Android app sandbox、SQLite、shared_prefs、iOS plist/backup。

## 本机工具清单（按使用时机）

### 首轮常用

| 工具 | 为什么放在首轮 |
|---|---|
| `file` | 先确认真实载体类型，决定后续分流 |
| `exiftool` | 图片、文档、媒体题的低成本首检 |
| `tshark` | PCAP 题最先看协议分布与明显流量 |
| `fls` / `icat` | 磁盘镜像题最先建立文件树与提取链 |
| `vol` | 内存镜像题确认进程、句柄、文件扫描 |
| `binwalk` | 固件/容器类题快速看嵌套结构 |

### 专项按需

- 恢复/雕刻：`foremost`、`scalpel`、`photorec`、`testdisk`
- 隐写：`zsteg`、`zsteg-mask`、`zsteg-reflow`、`zpng`、`steghide`、`zbarimg`
- PCAP 修复与抓取：`pcapfix`、`tcpdump`
- 文档与媒体：`pdfinfo`、`pdftotext`、`pdfimages`、`ffmpeg`、`olevba`、`pcodedmp`
- 文件系统深挖：`istat`、`blkcat`、`mmls`、`fsstat`、`debugfs` 等
- 运行中 Python 进程：`pyrasite`、`pyrasite-shell`、`pyrasite-memory-viewer`

### 当前未装 / 建议按需补装

- 当前没有必须补的基础工具。若后续经常做事件日志或注册表题，再单独评估是否补专用解析工具。

## 详细清单

### 文件提取/雕刻

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| binwalk | /usr/bin/binwalk | 固件/文件嵌入提取 | `binwalk -Me firmware.bin` |
| foremost | /usr/bin/foremost | 按文件头/尾/大小雕刻 | `foremost -t all -i image.dd` |
| scalpel | /usr/bin/scalpel | 基于配置的高性能文件雕刻 | `scalpel image.dd -o out/` |
| photorec | /usr/bin/photorec | 按文件签名恢复删除文件 | `photorec image.dd` |
| testdisk | /usr/bin/testdisk | 恢复删除分区、修复分区表 | `testdisk image.dd` |

### 磁盘/文件系统取证 (Sleuth Kit)

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| fls | /usr/bin/fls | 列出文件/目录（含已删除） | `fls -r image.dd` |
| icat | /usr/bin/icat | 按 inode 提取文件内容 | `icat image.dd 12345` |
| istat | /usr/bin/istat | 查看 inode 元数据 | `istat image.dd 12345` |
| blkcat | /usr/bin/blkcat | 按块号提取数据 | `blkcat image.dd 1000` |
| mmls | /usr/bin/mmls | 列出分区布局 | `mmls image.dd` |
| fsstat | /usr/bin/fsstat | 文件系统统计信息 | `fsstat image.dd` |
| blkcalc | /usr/bin/blkcalc | 文件系统块号与磁盘扇区映射 | `blkcalc -u 64 image.dd` |
| debugfs | /usr/sbin/debugfs | ext2/3/4 调试与恢复 | `debugfs -w image.dd` |
| e2fsck | /usr/sbin/e2fsck | ext2/3/4 文件系统检查修复 | `e2fsck -y image.dd` |
| dd | /usr/bin/dd | 磁盘/分区原始读写 | `dd if=image.dd bs=512 skip=2048` |
| losetup | /usr/sbin/losetup | 将文件映射为块设备 | `losetup -f image.dd` |

### 隐写/元数据

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| zsteg | /usr/local/bin/zsteg | PNG/BMP 位面隐写检测 | `zsteg -a image.png` |
| zsteg-mask | /usr/local/bin/zsteg-mask | 生成/测试 zsteg 掩码 | `zsteg-mask ...` |
| zsteg-reflow | /usr/local/bin/zsteg-reflow | 对像素流重新排布后再做隐写分析 | `zsteg-reflow ...` |
| zpng | /usr/local/bin/zpng | zsteg 配套 PNG 处理工具 | `zpng ...` |
| steghide | /usr/bin/steghide | JPEG/BMP/WAV 隐写嵌入/提取 | `steghide extract -sf image.jpg` |
| exiftool | /usr/bin/exiftool | 读取/写入文件元数据 | `exiftool suspicious.jpg` |
| zbarimg | /usr/bin/zbarimg | 二维码/条形码解码 | `zbarimg qrcode.png` |

### 网络/PCAP

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| tshark | /usr/bin/tshark | 命令行协议分析器 | `tshark -r capture.pcap -Y "http"` |
| tcpdump | /usr/bin/tcpdump | 网络抓包/过滤 | `tcpdump -r capture.pcap -A` |
| pcapfix | /usr/bin/pcapfix | 修复损坏的 PCAP 文件 | `pcapfix -d corrupted.pcap` |

### 密码/哈希

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| john | /usr/sbin/john | 密码哈希破解 | `john --wordlist=rockyou.txt hash.txt` |
| keepass2john | /usr/sbin/keepass2john | KeePass 数据库提取哈希 | `keepass2john db.kdbx > hash.txt` |
| openssl | /usr/bin/openssl | 加解密/证书/PKI | `openssl enc -d -aes-256-cbc -in enc.bin` |

### 音视频/图像

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| ffmpeg | /usr/bin/ffmpeg | 音视频转换/频谱图生成 | `ffmpeg -i audio.wav -lavfi showspectrumpic spec.png` |
| convert | /usr/bin/convert | ImageMagick 图像格式转换 | `convert image.png image.jpg` |
| magick | /usr/bin/magick | ImageMagick 7 统一入口 | `magick convert input.png output.jpg` |
| identify | /usr/bin/identify | 查看图像属性 | `identify -verbose image.png` |
| compare | /usr/bin/compare | 图像差异比较 | `compare a.png b.png diff.png` |

### PDF 工具

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| pdfinfo | /usr/bin/pdfinfo | PDF 元数据查看 | `pdfinfo document.pdf` |
| pdftotext | /usr/bin/pdftotext | PDF 文本提取 | `pdftotext document.pdf -` |
| pdfimages | /usr/bin/pdfimages | PDF 嵌入图像提取 | `pdfimages -j document.pdf out` |

### 压缩/归档

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| 7z | /usr/bin/7z | 通用压缩/解压 | `7z x archive.7z` |
| tar | /usr/bin/tar | tar 归档 | `tar xf archive.tar.gz` |
| gzip | /usr/bin/gzip | gzip 压缩 | `gzip -d file.gz` |
| unsquashfs | /usr/bin/unsquashfs | squashfs 文件系统解包 | `unsquashfs firmware.squashfs` |

### 通用

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| strings | /usr/bin/strings | 提取可打印字符串 | `strings file | grep -i flag` |
| file | /usr/bin/file | 识别文件真实类型 | `file unknown.bin` |
| git | /usr/bin/git | 版本控制/仓库取证 | `git log --all` |
| apktool | /usr/bin/apktool | APK 反编译 | `apktool d app.apk` |
| perl | /usr/bin/perl | 脚本/文本处理 | `perl -pe ...` |

### Ruby Gem

| 工具 | 路径 | 版本 | 功能 | 典型用法 |
|---|---|---|---|---|
| zsteg | /usr/local/bin/zsteg | 0.2.14 | PNG/BMP 位面隐写综合检测 | `zsteg -a image.png` |

### ctf-tools conda 环境 (Python 包)

| 包名 | 版本 | 功能 | 典型用法 |
|---|---|---|---|
| volatility3 | 2.27.0 | 内存取证框架（ctf-tools 中命令为 `vol`） | `vol -f memory.dmp windows.pslist` |
| oletools | 0.60.2 | Office 文档宏/结构分析 | `olevba suspicious.doc` |
| pcodedmp | 1.2.6 | VBA p-code 提取，适合宏源码被篡改或 stomped 时补看真实逻辑 | `pcodedmp suspicious.doc` |
| msoffcrypto-tool | 6.0.0 | 加密 Office 文档解密 | `msoffcrypto-tool encrypted.docx decrypted.docx` |
| Pillow | 11.3.0 | Python 图像处理库 | `Image.open('stego.png')` |
| numpy | 2.4.4 | 数值计算/多维数组 | `np.array(pixels)` |
| scipy | 1.17.1 | 科学计算/信号处理 | `scipy.signal.spectrogram(audio)` |
| matplotlib | 3.10.8 | 数据可视化/绘图 | `plt.imshow(spectrogram)` |
| scapy | 2.7.0 | 数据包构造与解析 | `scapy.rdpcap('capture.pcap')` |
| pycryptodome | 3.17 | 加密算法库 | `from Crypto.Cipher import AES` |
| uncompyle6 | 3.9.3 | Python 2.7-3.8 bytecode 反编译 | `uncompyle6 compiled.pyc` |
| olefile | 0.47 | OLE 复合文档解析 | `olefile.isOleFile('doc.xls')` |
| pyzbar | 0.1.9 | Python 条码/二维码解码绑定 | `from pyzbar.pyzbar import decode` |
| zxing-cpp | 3.0.0 | 更广谱的条码/二维码识别库 | `import zxingcpp` |

### 独立工具

| 工具 | 路径 | 功能 | 典型用法 |
|---|---|---|---|
| pycdc | /home/kali/pycdc/build/pycdc | Python 3.9+ bytecode 反编译 | `pycdc file.pyc` |
| pyrasite | /usr/local/bin/pyrasite | 向运行中的 Python 进程注入代码 | `pyrasite PID payload.py` |
| pyrasite-shell | /usr/local/bin/pyrasite-shell | 连接运行中的 Python 进程交互 shell | `pyrasite-shell PID` |
| pyrasite-memory-viewer | /usr/local/bin/pyrasite-memory-viewer | 查看运行中 Python 进程对象/内存 | `pyrasite-memory-viewer PID` |
