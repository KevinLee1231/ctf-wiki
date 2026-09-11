---
type: tooling
tags: [stego, tooling, tools, environment, media]
skills: [ctf-stego]
---

# Stego Tooling

本页服务媒体、文档、文本、QR、场景或协议中的隐藏载荷。普通可逆编码见 [crypto-tooling.md](crypto-tooling.md)，事件与证据恢复见 [forensics-tooling.md](forensics-tooling.md)。

## 平台与现存工具

图片、位面、频谱数组、帧差、QR 重组与文档脚本优先在 Windows 完成，使用 `D:/文档/新建文件夹/venv/Scripts/python.exe`（3.14.3）。旧 Conda 图像、QR 和 OCR 包不再作为入口。

| Windows 工具 | 当前状态 | 用途 |
|---|---|---|
| `Pillow` | venv `12.2.0` | 图像模式、通道、像素、裁剪与重排 |
| `numpy` / `scipy` | venv `2.4.4` / `1.17.1` | 位面、阈值、FFT、滤波与相关 |
| `matplotlib` | venv `3.11.0` | 位图、频谱、时序和散点输出 |
| `PyMuPDF` | venv `1.27.2.3` | PDF 文本、嵌图与附件；更多文档工具见 Forensics 页 |
| StegSolve | `D:/CTF工具/stegsolve.jar`，文件存在 | 手工通道、位面和图像变换；GUI 未经启动不能当作已验证会话 |

| 已有 WSL 工具 | 位置与版本 | 用途 |
|---|---|---|
| zsteg / zpng | `/usr/local/bin/zsteg`、`/usr/local/bin/zpng`；Ruby gem `0.2.14` / `0.4.6` | PNG/BMP 位面检查与 PNG 结构辅助；同目录还有 zsteg-mask、zsteg-reflow |
| steghide | `/usr/bin/steghide`；APT `0.5.1+git20240220-2` | 支持容器的信息与载荷提取 |
| zbarimg / qrencode | `/usr/bin/zbarimg`、`/usr/bin/qrencode`；APT `0.23.93-9+b4` / `4.1.1-2+b2` | 已有条码解码与 QR 生成 |
| exiftool | `/usr/bin/exiftool`；APT `13.55+dfsg-1` | 媒体、文档元数据 |
| ffmpeg / sox | `/usr/bin/ffmpeg`、`/usr/bin/sox`；APT `7:8.1.2-2+b3` / `14.7.1.2+ds1-1` | 音视频转换、帧与频谱输出 |
| qsstv | `/usr/bin/qsstv`；APT `9.5.8-6` | SSTV GUI，需实际音频/显示环境；未验证收音链 |

当前没有可用的 pytesseract/Tesseract、OpenCV、Python QR 绑定或 Piet 解释器入口。普通像素与信号工作先使用上表；需要 OCR、新条码格式或专门 esolang 解释器时，先评估 Windows 最小方案，不自动补回 WSL。

## 从 pwsh 调用

题目路径与输出按实际任务替换。WSL 工具属于 APT/Ruby 层，不套 Conda：

```pwsh
& "D:/文档/新建文件夹/venv/Scripts/python.exe" -c 'from PIL import Image; im=Image.open("D:/题目路径/image.png"); print(im.format, im.mode, im.size); print(im.getextrema())'
& "D:/文档/新建文件夹/venv/Scripts/python.exe" "D:/题目路径/extract_bitplanes.py"
wsl -d kali-linux --exec /usr/local/bin/zsteg -a "/mnt/d/题目路径/image.png"
wsl -d kali-linux --exec /usr/bin/zbarimg --quiet "/mnt/d/题目路径/qr.png"
wsl -d kali-linux --exec /usr/bin/qrencode -o "/mnt/d/题目路径/qr-output.png" "known test text"
wsl -d kali-linux --exec /usr/bin/steghide info "/mnt/d/题目路径/image.jpg"
wsl -d kali-linux --exec /usr/bin/exiftool "/mnt/d/题目路径/image.png"
wsl -d kali-linux --exec /usr/bin/sox "/mnt/d/题目路径/audio.wav" -n spectrogram -o "/mnt/d/题目路径/spectrum.png"
wsl -d kali-linux --exec /usr/bin/ffmpeg -i "/mnt/d/题目路径/video.mp4" -frames:v 1 "/mnt/d/题目路径/first-frame.png"
```

`D:/题目路径` 对应 `/mnt/d/题目路径`。`zsteg -h` 可查看帮助；其 `--version` 退出码不能单独用于判定工具损坏。`-a` 会尝试多种位面组合，大图先裁剪或限制参数。steghide 若询问口令，应按题目已知条件处理，不自动进入爆破。

StegSolve 需要手工查看图像时可在 Windows 运行：

```pwsh
& "C:/Program Files/Eclipse Adoptium/jdk-21.0.10.7-hotspot/bin/java.exe" -jar "D:/CTF工具/stegsolve.jar"
```

## 失败处理与下一跳

- 位面噪声或大量假阳性：核对模式、透明通道、像素顺序、位序与 payload 长度；见 [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)。
- 多图、帧差或通道重组：见 [media-channel-bitplane-and-frame-difference-extraction.md](media-channel-bitplane-and-frame-difference-extraction.md)。
- QR 不能识别：先验证定位图案、quiet zone、网格与方向，再查 [qr-and-structured-symbol-reassembly.md](qr-and-structured-symbol-reassembly.md)，不要直接加装一组解码器。
- 音频看不出结构：确认采样率、声道、符号率和调制，再查 [audio-spectrum-and-symbol-decoding.md](audio-spectrum-and-symbol-decoding.md)。
- PDF/视频对象或隐藏帧：见 [video-document-and-media-stego.md](video-document-and-media-stego.md)；GUI 文件存在不代表已完成媒体解码。
- WSL 新增、升级或重装须先说明用途、Linux 必要性、方式、目标及依赖影响并取得明确同意。
