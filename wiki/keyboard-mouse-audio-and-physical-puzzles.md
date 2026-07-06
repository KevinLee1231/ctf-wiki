---
type: family
tags: [forensics, misc, hid, audio, video, physical-signal, family]
skills: [ctf-forensics, ctf-misc]
raw:
  - ../raw/misc/keyboard-mouse-audio-and-physical-puzzles.md
updated: 2026-06-18
---

# Keyboard, Mouse, Audio and Physical Puzzles

## 作用边界

本页是物理/外设信号恢复 family。题目依赖人机输入轨迹、物理运动、音频/视频可视信号或外设协议还原后的事件序列，而不是传统编码本身。典型目标是把“动作记录”还原成文字、图形、坐标序列或频率序列。

边界在于“动作/信号语义还原”：如果证据仍停留在 USB URB、HID report、Bluetooth RFCOMM 或 report descriptor 层，先转 [peripheral-capture.md](peripheral-capture.md) 解析字段；本页处理解析后的坐标、按键、频率、帧和物理轨迹。

## 识别信号

- 附件包含 USB PCAP、鼠标/键盘 HID、屏幕键盘视频、打印机/机械运动视频、音频 WAV/MP3。
- 数据看起来像相对位移、点击事件、频谱、按键时序或物理轨迹。
- flag 不是直接藏在文件中，而是由轨迹画出来、敲出来、唱出来或打印出来。
- 同一事件有明显 pen-up/pen-down、click/release、frame interval 或音调持续时间。

## 最小证据

- 能从原始数据提取时间序列：鼠标 dx/dy、键盘 keycode、音频频率、视频中喷头/LED/光点坐标。
- 至少可视化一小段轨迹或频谱，证明数据承载信息而不是噪声。
- 能区分移动、点击、抬笔、静音、空闲模式等状态。

## 分流流程

1. 先转成表格：时间戳、事件类型、数值字段、状态字段。
2. USB HID：若仍是 PCAP/report，先用 [peripheral-capture.md](peripheral-capture.md) 解析字段；本页继续处理相对位移累加、click falling edge 取点和屏幕键盘映射。
3. 音频：先生成频谱图；若像调制信号再转 forensics/RF 页面，若像音符则提主频映射字符。
4. 视频/物理：用 OpenCV 跟踪关键点，过滤真正打印/按压/亮灯帧，再画 2D 轨迹或时序。
5. 对输出做 OCR/QR/人工读图，必要时旋转、缩放、反色。

## 信号路线分流

| 信号形态 | 下一跳判断 |
|---|---|
| USB mouse 轨迹 | `dx/dy` 已从 report 中提取后，累加成绝对坐标；点击边沿决定有效点。 |
| Audio spectrogram | `sox audio.wav -n spectrogram` 先看是否有可视文本或频率编码。 |
| 3D printer video | 跟踪喷头/床位置，过滤挤出动作，把轨迹投影成文字。 |
| LED/Morse/按键时序 | 用 on/off 持续时间聚类成点、划、分隔。 |

## 常见陷阱

- 只看文件 metadata，忽略真正信息在时间序列中。
- 鼠标相对位移没有累加，导致点云完全不成图。
- 视频没有过滤非打印/非按压帧，轨迹被空走路径污染。
- 频谱图生成参数太粗，短音或高频细节被抹掉。

## 关联技巧

- [peripheral-capture.md](peripheral-capture.md)
- [signals-and-hardware.md](signals-and-hardware.md)
- [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md)
- [3d-printing.md](3d-printing.md)
- [rf-sdr.md](rf-sdr.md)
- [video-document-and-media-stego.md](video-document-and-media-stego.md)

## 原始资料

- [keyboard-mouse-audio-and-physical-puzzles.md](../raw/misc/keyboard-mouse-audio-and-physical-puzzles.md)
