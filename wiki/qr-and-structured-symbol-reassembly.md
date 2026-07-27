---
type: technique
tags: [crypto, stego, qr, barcode, reassembly, error-correction]
skills: [ctf-crypto, ctf-stego, ctf-forensics]
raw:
  - ../raw/crypto/encodings-qr-and-esolangs.md
  - ../raw/stego/image-bitplane-qr-and-jpeg-stego.md
updated: 2026-07-27
---

# QR and Structured-Symbol Reassembly

## 适用场景

QR/条码/网格符号被分片、旋转、反色、遮挡、打乱或嵌入媒体，需利用 finder、timing、format 和 error-correction 结构恢复可解码矩阵。

## 识别信号

- 出现方格、三角定位块、固定行列节奏或有限颜色符号。
- 多张图片/帧提供互补碎片，单张无法识别。
- 视觉内容看似随机，但尺寸接近标准 QR module 数。

## 最小证据

- 确认模块尺寸、边界 quiet zone、颜色极性和候选版本。
- 定位 finder/timing/alignment pattern，证明拼接符合结构。
- 解码结果可由纠错状态、payload 格式或校验复验。

## 解法骨架

1. 二值化并估计 module grid，不直接对缩放截图调用扫码器。
2. 用定位与 timing pattern 约束旋转、镜像、分片顺序和极性。
3. 合并多帧/多层证据，未知模块保留 mask 而非随意填值。
4. 恢复 quiet zone 后用多个解码器交叉验证并保存最终矩阵。

## 关键变体

- 碎片拼图：结构模式比图像相似度更可靠。
- 多帧/多层 QR：按时间、通道或 bitplane 合并。
- 自定义二维码：先恢复 symbol mapping，再解析 payload。

## 常见陷阱

- 截图缩放造成非整数 module，直接解码持续失败。
- 忽略镜像、反色和 quiet zone。
- 为通过扫码器手工涂改，却没有结构证据。

## 关联技巧

- [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md)
- [image-bitplane-qr-and-jpeg-stego.md](image-bitplane-qr-and-jpeg-stego.md)
- [layered-encoding-and-symbol-mapping-recovery.md](layered-encoding-and-symbol-mapping-recovery.md)

## 原始资料

- [encodings-qr-and-esolangs.md](../raw/crypto/encodings-qr-and-esolangs.md)
- [image-bitplane-qr-and-jpeg-stego.md](../raw/stego/image-bitplane-qr-and-jpeg-stego.md)
