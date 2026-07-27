---
type: technique
tags: [stego, image, video, bitplane, channel, frame-difference]
skills: [ctf-stego]
raw:
  - ../raw/stego/image-bitplane-qr-and-jpeg-stego.md
  - ../raw/stego/video-document-and-media-stego.md
updated: 2026-07-27
---

# Media-Channel, Bitplane and Frame-Difference Extraction

## 适用场景

图像/视频的信息藏在颜色通道、bitplane、alpha、像素遍历顺序、帧差或时间维度；可见画面只是载体，需系统枚举通道与排序后重组字节/图像。

## 识别信号

- 通道/bitplane 熵、直方图或帧间差异异常。
- PNG/BMP/GIF/JPEG 解码后像素中存在周期性低位。
- 多帧静态背景上有微弱变化、闪烁或移动轨迹。

## 最小证据

- 固定解码后的像素格式、尺寸、通道顺序和帧率。
- 对每种提取记录 channel/bit/order/packing。
- 输出通过 magic、QR 结构、文本或独立视觉结果验证。

## 解法骨架

1. 先拆 RGB/A、bitplane、行列扫描和帧序列。
2. 用熵、周期、已知前缀和结构评分筛选候选。
3. 视频计算相邻/背景帧差并累积轨迹或二值 mask。
4. 按 MSB/LSB 与 offset 重组字节，保存参数和产物。

## 关键变体

- Pixel/channel LSB 与 bitplane。
- Palette/alpha/metadata-independent 隐写。
- Video frame difference、时间采样和运动轨迹。

## 常见陷阱

- 混淆像素通道顺序和 bit order。
- 对压缩 JPEG 像素直接套无损 LSB 假设。
- 只展示肉眼图，没有记录可复现提取参数。

## 关联技巧

- [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)
- [video-document-and-media-stego.md](video-document-and-media-stego.md)
- [qr-and-structured-symbol-reassembly.md](qr-and-structured-symbol-reassembly.md)

## 原始资料

- [image-bitplane-qr-and-jpeg-stego.md](../raw/stego/image-bitplane-qr-and-jpeg-stego.md)
- [video-document-and-media-stego.md](../raw/stego/video-document-and-media-stego.md)
