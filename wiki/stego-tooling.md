---
type: tooling
tags: [stego, tooling, tools, environment, media]
skills: [ctf-stego]
---

# Stego Tooling

本页是 `ctf-stego` 方向本机工具信息的唯一权威来源，维护当前安装状态、版本、路径、环境、完整调用、适用边界和失败处理。`ctf-stego/SKILL.md` 只说明何时选择某类工具或知识页，不复制本页细节。

本页只描述当前真实状态；实际环境与本文不一致时，直接修正文中现状，不在本页累积核验记录或旧版本历史。

## 工具选择边界

- 先保存原始文件和 hash，避免转码、重采样或元数据写入破坏隐藏通道。
- 工具必须由具体异常触发：通道/bitplane、尾部/嵌入、QR 网格、帧差、声道/频谱或文档对象；无异常时不堆自动扫描器。
- 文件修复、删除恢复和正常会话重组转 Forensics；普通可逆编码转 Crypto；RF/IQ 与物理信号转 Hardware/Embedded。

## 完整调用约定

系统工具通过 WSL 绝对路径调用；Python 图像/信号脚本使用 `ctf-tools`。所有命令从 `pwsh` 发起：

```pwsh
wsl /usr/bin/file /path/to/carrier.bin
wsl /usr/bin/exiftool /path/to/carrier.png
wsl /usr/local/bin/zsteg -a /path/to/image.png
wsl /usr/bin/zbarimg /path/to/reassembled-qr.png
wsl /usr/bin/ffmpeg -i /path/to/audio.wav -lavfi showspectrumpic /path/to/spectrum.png
wsl /home/kali/miniforge3/bin/conda run --no-capture-output -n ctf-tools python /path/to/stego_analysis.py
```

## 当前状态与路径

| 工具 | 当前状态/版本 | 路径或环境 | 何时使用 | 完整调用 |
|---|---|---|---|---|
| `file` | 可用，5.47 | `/usr/bin/file`，WSL system | 真实格式与容器首检 | 见上方命令 |
| `xxd` | 可用，Vim xxd 9.2.0524 | `/usr/bin/xxd`，WSL system | chunk、header、尾部和字节序检查 | `wsl /usr/bin/xxd -g 1 /path/to/carrier.bin` |
| `exiftool` | 可用，13.55 | `/usr/bin/exiftool`，WSL system | 图片、文档和媒体 metadata | 见上方命令 |
| `zsteg` | 可用，0.2.14 | `/usr/local/bin/zsteg`，Ruby Gem | PNG/BMP 通道和 bitplane 检查 | 见上方命令 |
| `steghide` | 可用，入口报告 0.6.0 | `/usr/bin/steghide`，WSL system | JPEG/BMP/WAV 常见嵌入载荷 | `wsl /usr/bin/steghide info /path/to/carrier.jpg` |
| `zbarimg` | 可用，0.23.93 | `/usr/bin/zbarimg`，WSL system | QR/条码完整性和重组验证 | 见上方命令 |
| `ffmpeg` | 可用，8.1.2 | `/usr/bin/ffmpeg`，WSL system | 帧、声道和频谱的无损拆分 | 见上方命令 |
| `sox` | 可用，14.7.1.2 | `/usr/bin/sox`，WSL system | 音频通道、反转、滤波和 spectrogram | `wsl /usr/bin/sox /path/to/audio.wav -n spectrogram -o /path/to/spec.png` |
| `mutool` | 可用，1.27.0 | `/usr/bin/mutool`，WSL system | PDF 对象、文本、图片和增量结构 | `wsl /usr/bin/mutool info /path/to/document.pdf` |
| Python 图像/信号包 | Pillow 11.3.0、numpy 2.4.4、scipy 1.17.1、matplotlib 3.10.8、pyzbar 0.1.9、zxing-cpp 3.0.0 | WSL `ctf-tools` | 自定义像素顺序、帧差、频谱、网格和条码重组 | 见上方 Python 脚本命令 |
| Poppler CLI | `pdfinfo`、`pdftotext`、`pdfimages` 当前未安装 | WSL | PDF 元数据、文本和图片的替代提取链 | 当前先用 `mutool info/draw/extract`，不要调用不存在的入口 |

## 失败处理

- 自动隐写工具无命中：回到通道、bit、扫描顺序、端序、帧序、频谱和对象结构，不把“未命中”当作无隐藏信息证明。
- QR 解码失败：先检查 finder、网格尺寸、quiet zone、mask、纠错和碎片顺序；保留每次重组图像。
- 音视频拆分后信息消失：确认没有有损转码、重采样或声道混合，优先从原始载体重新导出。
- PDF 需要 Poppler 特定行为：先验证 `mutool` 是否足够；确需缺失工具时再按全局授权规则安装，并更新本页当前状态。

## 关联知识页

- [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)
- [media-channel-bitplane-and-frame-difference-extraction.md](media-channel-bitplane-and-frame-difference-extraction.md)
- [qr-and-structured-symbol-reassembly.md](qr-and-structured-symbol-reassembly.md)
- [audio-spectrum-and-symbol-decoding.md](audio-spectrum-and-symbol-decoding.md)
- [video-document-and-media-stego.md](video-document-and-media-stego.md)
