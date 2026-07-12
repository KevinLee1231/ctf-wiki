---
type: family
tags: [forensics, family, signal, hardware, side-channel, bus]
skills: [ctf-hardware-embedded, ctf-forensics, ctf-reverse]
raw:
  - ../raw/hardware-embedded/signals-and-hardware.md
updated: 2026-06-12
---

# Signals and Hardware

## 作用边界

本页是物理信号、显示链路、总线协议、输入设备和侧信道取证 family。它处理的不是普通图片/音频隐写，而是从采样波形、逻辑分析仪、视频帧、RF 文件、键盘声学、功耗或硬件协议中恢复可读数据。

如果附件已经是常规 PCAP、磁盘或内存镜像，先走对应 forensics family；只有证据需要还原采样率、总线帧、物理编码或设备状态时进入本页。

## 识别信号

- 附件是 Saleae/logic capture、WAV 采样、`.sub`、视频拍摄的 LED/键盘、显示链路数据、USB/MIDI、I2C/UART/SPI 或 power trace。
- 数据看起来像屏幕扫描线、TMDS、8b/10b、LFSR scrambler、按键事件、摩斯、FSK/ASK/OOK、UART 起止位或总线时序。
- 题目提示硬件型号、采样率、baud rate、GPIO、Flipper Zero、VGA/HDMI/DisplayPort、side-channel 或 keyboard acoustic。
- 普通文件 carving 无结果，但波形/帧/事件重建后可出现文本、图片、按键或密钥。

## 最小证据

- 原始采样参数：采样率、通道数、时钟、baud、bit order、voltage threshold、帧率或像素时序。
- 物理层到逻辑层的映射：边沿、符号、起止位、编码表、scrambler、扫描顺序或按键矩阵。
- 一个可复算中间产物：解出的字节流、帧图像、按键序列、频谱峰、协议包或 side-channel trace alignment。
- 噪声和同步策略：阈值、重采样、clock recovery、去抖、重复采样或多数投票。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| VGA/HDMI/DisplayPort | 像素时钟、同步信号、TMDS/8b10b 和 scrambler 是否可恢复 | 重建帧图像，必要时转图像隐写页 |
| UART/I2C/SPI/USB MIDI | 采样率、baud、bit order、起止位和地址/命令字段 | [peripheral-capture.md](peripheral-capture.md) |
| Flipper `.sub` / RF | 调制方式、频率、pulse 宽度和重复帧 | [rf-sdr.md](rf-sdr.md) |
| 功耗/侧信道 trace | trace 对齐、触发点、明文/密文和泄露模型 | 转 crypto 或 reverse 实现恢复 |
| 键盘声学/LED/视频摩斯 | 帧率、按键事件、LED 闪烁节奏和同步点 | [keyboard-mouse-audio-and-physical-puzzles.md](keyboard-mouse-audio-and-physical-puzzles.md) |
| CD/audio disc image | 轨道、sector、subchannel、音频采样和隐藏数据 | [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md) |
| 打孔卡/旧设备 OCR | 物理位置到字符编码表的映射 | [video-document-and-media-stego.md](video-document-and-media-stego.md) |

## 合并与拆分结论

- 保留为 family：raw 覆盖显示链路、总线、RF、功耗、键盘声学、音频和旧硬件介质，单一 technique 无法覆盖。
- 不合并进 `audio-frequency-and-archive-stego.md`：本页强调物理/协议还原，音频页强调音频隐写和频域/压缩包恢复。
- 不合并进 `hardware-isa-bootloader-and-kvm.md`：该页是 reverse/ISA 运行环境，本页是 forensics 信号恢复。

## 常见误判

- 只用文件 magic 判断，忽略采样率、时钟和通道语义。
- UART/I2C 解码不先确认 bit order 和阈值，得到乱码后误判为加密。
- 显示信号只看截图，不还原同步和像素顺序。
- 侧信道 trace 没有对齐和噪声处理，直接相关性分析结论不稳定。

## 关联页面

- [cross-domain-forensics-technique-map.md](cross-domain-forensics-technique-map.md)
- [peripheral-capture.md](peripheral-capture.md)
- [rf-sdr.md](rf-sdr.md)
- [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md)
- [keyboard-mouse-audio-and-physical-puzzles.md](keyboard-mouse-audio-and-physical-puzzles.md)
- [video-document-and-media-stego.md](video-document-and-media-stego.md)
- [forensics-tooling.md](forensics-tooling.md)

## 原始资料

- [signals-and-hardware.md](../raw/hardware-embedded/signals-and-hardware.md)
