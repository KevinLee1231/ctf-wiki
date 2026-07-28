---
type: technique
tags: [stego, forensics, audio, spectrum, dtmf, sstv]
skills: [ctf-stego, ctf-forensics]
raw:
  - ../raw/stego/audio-frequency-and-archive-stego.md
  - ../raw/forensics/network-covert-auth-and-reassembly.md
  - ../raw/stego/ACTF2022-ffsk-wp.md
updated: 2026-07-28
---

# Audio Spectrum and Symbol Decoding

## 适用场景

音频通过固定频率、DTMF、Morse、SSTV、FSK、超声或声道差异编码符号；需要先确定时频参数，再将频率/时长序列离散成字符或图像。

## 识别信号

- Spectrogram 出现稳定水平线、字符轮廓或周期 burst。
- 音调集中在 DTMF/FSK/Morse/SSTV 等有限频率集合。
- 左右声道、相位或低位数据存在明显差分。

## 最小证据

- 记录采样率、声道、窗口、hop、频率 bin 和时间单位。
- 已知前导/同步/校验能验证符号映射。
- 解码结果可从原波形按参数重复生成。

## 解法骨架

1. 检查 WAV/container、采样率、声道和原始 PCM。
2. 生成多尺度 spectrogram，定位载波和符号边界。
3. 对每段估计主频/能量/时长并映射协议符号。
4. 处理同步、反相、字节序和纠错，输出文字/图像并复验。

## 关键变体

- DTMF/Morse/FSK。
- SSTV/图像类调制。
- Stereo difference、phase 或 PCM low-byte channel。

## 常见陷阱

- 仅凭一张 spectrogram 肉眼读字。
- 采样率/窗口选择错误导致频率漂移。
- 将压缩编码伪影误判为载波。

## 关联技巧

- [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md)
- [signals-and-hardware.md](signals-and-hardware.md)
- [peripheral-event-and-coordinate-reconstruction.md](peripheral-event-and-coordinate-reconstruction.md)

## 原始资料

- [audio-frequency-and-archive-stego.md](../raw/stego/audio-frequency-and-archive-stego.md)
- [network-covert-auth-and-reassembly.md](../raw/forensics/network-covert-auth-and-reassembly.md)
- [ACTF2022-ffsk-wp](../raw/stego/ACTF2022-ffsk-wp.md)
