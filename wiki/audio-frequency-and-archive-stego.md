---
type: family
tags: [stego, forensics, family, audio, frequency, archive]
skills: [ctf-stego, ctf-forensics]
raw:
  - ../raw/stego/audio-frequency-and-archive-stego.md
  - ../raw/forensics/WMCTF2025-voice-hacker-wp.md
updated: 2026-07-28
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
| [UMDCTF2017-synesthesia-2-wp](../raw/stego/UMDCTF2017-synesthesia-2-wp.md) | 高位宽 PCM 的不同字节可以承载彼此独立的信号。正常播放器按完整 32 bit little-endian 整数解释样本，最低字节中的小信号会被高位声音掩盖；把该字节单独提升为音频后，隐藏频谱才会显现。二维码谱线还需要按其几何方向做有限形态学修复。 |
| [UMDCTF2019-secretsounds-wp](../raw/stego/UMDCTF2019-secretsounds-wp.md) | 8 位 PCM 的符号位翻转可以让任意字节流听起来像音频数据，同时用固定异或恢复。分析时应读取容器声明的数据偏移，并检查简单变换后的 magic。恢复出的可执行文件也不必直接运行；若尾部是已知归档格式，静态列目录和解包更安全。 |
| [UMDCTF2021-coldplays-flags-wp](../raw/stego/UMDCTF2021-coldplays-flags-wp.md) | 音频在本题同时充当宿主文件和语义线索。`binwalk` 负责发现追加 ZIP，时间戳则要求回到歌曲内容取词。外部歌曲信息不是解题的唯一载体：关键时间点、拼接规则和最终口令均已在这里展开，按照原音频即可独立复现。 |
| [WMCTF2022-hilbert-wave-wp](../raw/stego/WMCTF2022-hilbert-wave-wp.md) | 本题核心是识别“一维音频数据其实是二维图像数据”，再根据题名使用 Hilbert 曲线逆映射恢复空间顺序。$49152=128\times128\times3$、三声道、采样值不超过 255 是判断 RGB 图片的关键证据。 |
| [WMCTF2022-spider-man-wp](../raw/stego/WMCTF2022-spider-man-wp.md) | 本题核心是不要停在游戏内可见 flag。触碰模型后的音效尾部出现杂音，结合开发者工具中 `hit-crystal` 资源，说明真正数据藏在音频文件里；识别信号是：VR 场景中的物理 flag 过于直接、触发音效尾部异常、资源名和交互事件能对应到 `hit-crystal`、文件扩展名与实际内容不一致。 |
| [WMCTF2023-money-left-me-broken-wp](../raw/stego/WMCTF2023-money-left-me-broken-wp.md) | 先用 Arnold 逆变换恢复视频，再用原始音频作差提取 DCT watermark；视频画面呈周期性扭曲、题名或画面提示原始素材来源、音频存在噪声时，应考虑“视频置乱 + 音频水印”双层隐写。 |

## Technique 下一跳

| 首轮判断 | 具体 technique |
|---|---|
| 固定频率、DTMF/Morse/SSTV/FSK 或声道差异编码符号 | [audio-spectrum-and-symbol-decoding.md](audio-spectrum-and-symbol-decoding.md) |
| 信息位于声道/bitplane/时间帧差等媒体通道 | [media-channel-bitplane-and-frame-difference-extraction.md](media-channel-bitplane-and-frame-difference-extraction.md) |
| 音频尾部/容器内嵌归档或结构损坏 | [archive-structure-repair-and-stream-carving.md](archive-structure-repair-and-stream-carving.md) |

## 合并与拆分结论

- 保留为 family：audio raw 横跨频域、双音、SSTV、LSB、元数据、语音特征和 archive。
- 不合并进 `signals-and-hardware.md`：本页从已成音频文件和频域证据出发，硬件页从采样/总线/物理层出发。
- DTMF/SSTV/频率符号已归入统一的音频解码 technique，媒体通道与归档恢复分别转到相邻 technique；本页继续承担三类证据的二级分流。

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
