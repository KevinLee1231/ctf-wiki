---
type: family
tags: [stego, forensics, family, audio, frequency, archive]
skills: [ctf-stego, ctf-forensics]
raw:
  - ../raw/stego/audio-frequency-and-archive-stego.md
  - ../raw/forensics/WMCTF2025-voice-hacker-wp.md
updated: 2026-07-06
---

# Audio, Frequency and Archive Stego

## 作用边界

本页是音频、频域、声学编码、音频隐写和轻量 archive 嵌套 family。它处理 FFT/频谱、SSTV、DTMF、双音键盘、跨声道 LSB、音轨差分、音频元数据、DeepSound、波形二进制和部分与音频混合的压缩包线索。

如果证据是硬件采样、UART 或 RF，先看 [signals-and-hardware.md](signals-and-hardware.md)。如果主体是视频/文档/图像容器，转 [video-document-and-media-stego.md](video-document-and-media-stego.md) 或图像 family。

## 识别信号

- 附件是 WAV/MP3/FLAC、频谱图、SSTV 音频、双音、多个声道、音轨集合或音频工具导出的 archive。
- 频谱、相位、声道差、采样率、音高、节奏、DTMF 键盘频率或 LSB 位平面出现异常。
- 题目声称语音认证，但源码或后端只比较 wav 长度、采样率、振幅、样本数或相似度阈值。
- 普通播放无结果，但反转、降噪、频谱、声道分离或元数据检查能出现结构。

## 最小证据

- 采样率、声道数、位深、时长、编码格式和是否有损压缩。
- 可复算中间层：频谱峰、DTMF 序列、SSTV 图、LSB bitstream、metadata 字段或解出的 archive。
- 对多声道/多音轨题，确认对齐方式、相位差和相减/叠加结果。
- 对语音认证题，确认真实比较特征，而不是按题面叙事盲做语音识别。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| 频谱/FFT 图案 | 频率轴、时间轴、窗长和峰值映射 | 读文本/QR/音符或转图像页 |
| SSTV | 模式、同步头、分辨率和是否红鲱鱼 | 解图后继续查 LSB/条码 |
| DTMF/双音 | 频率表是否标准，按键间隔和去噪阈值 | 转文本或验证码流程 |
| 多声道 LSB | 位平面、声道顺序和交叉拼接方式 | 还原 bitstream 再解码 |
| 多音轨差分 | 音轨是否对齐，是否需要反相/相减 | 提取差分残留 |
| 音频 metadata | 注释、octal/base、ID3、cue 或 hidden chunk | 转编码 family |
| DeepSound/密码 | 工具格式、密码线索和字典范围 | 跑工具前先备份原音频 |
| 声纹/语音认证 | 后端到底比什么特征 | 伪造满足特征的 wav |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-voice-hacker-wp](../raw/forensics/WMCTF2025-voice-hacker-wp.md) | 从 RTP 导出的参考音频可以用 TTS/语音克隆生成口令音频，但真正验收条件是后端比较的 wav 长度、振幅、采样数和相似度阈值。 |

## 合并与拆分结论

- 保留为 family：audio raw 横跨频域、双音、SSTV、LSB、元数据、语音特征和 archive。
- 不合并进 `signals-and-hardware.md`：本页从已成音频文件和频域证据出发，硬件页从采样/总线/物理层出发。
- 暂不拆 DTMF/SSTV/LSB 小页：当前页面作为音频二级分流更有查询价值。

## 常见误判

- 只听音频不看频谱、声道和元数据。
- 用有损重编码后的文件做 LSB/波形分析。
- DTMF 自定义频率被误按标准电话键盘解码。
- 语音认证题被叙事误导，没有看后端实际比较字段。

## 关联页面

- [signals-and-hardware.md](signals-and-hardware.md)
- [video-document-and-media-stego.md](video-document-and-media-stego.md)
- [keyboard-mouse-audio-and-physical-puzzles.md](keyboard-mouse-audio-and-physical-puzzles.md)
- [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md)
- [forensics-tooling.md](forensics-tooling.md)

## 原始资料

- [audio-frequency-and-archive-stego.md](../raw/stego/audio-frequency-and-archive-stego.md)
- [WMCTF2025-voice-hacker-wp](../raw/forensics/WMCTF2025-voice-hacker-wp.md)
