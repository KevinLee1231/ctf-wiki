# GreyCTF 2023 actualbeepboop

## 题目简述

附件是一段 48 kHz WAV 信号。GNU Radio 流图显示，原始字节先按 LSB-first 从 8 位重打包为 2 位符号，每个符号重复 960 个采样，再由 VCO 映射到四个音频频率，因此这是 50 baud 的 4-FSK 调制题。

## 解题过程

流图参数为中心频率 2210 Hz、相邻频率间隔 340 Hz。VCO 输入的偏置和幅度使四个符号分别对应约：

```text
符号 0 -> 2040 Hz
符号 1 -> 2380 Hz
符号 2 -> 2720 Hz
符号 3 -> 3060 Hz
```

采样率为 48000 Hz，波特率为 50，所以每个符号恰有 $48000/50=960$ 个采样。读取 WAV 单声道样本，以 960 点为一窗做 FFT，取正频率最大谱峰，并映射到最近的四个中心频率：

```python
tones = [2040, 2380, 2720, 3060]
symbols = []
for block in windows(samples, 960):
    spectrum = abs(rfft(block * hann(960)))
    peak = rfftfreq(960, 1 / 48000)[spectrum.argmax()]
    symbols.append(min(range(4), key=lambda i: abs(peak - tones[i])))
```

`repack_bits_bb(8, 2, ..., GR_LSB_FIRST)` 表示一个字节的低两位最先发送。每四个符号还原一个字节：

```python
byte = s0 | (s1 << 2) | (s2 << 4) | (s3 << 6)
```

解码文本前部有一段重复字符作为同步，随后 flag 出现两次。取其中一份得到：

```text
grey{b33pb00p_w1th_4_t0n3s_3509gfj09rfj09jg}
```

## 方法总结

确定数字调制信号时，应先从采样率、符号持续时间和离散频点恢复符号流，再处理位序和字节打包。这里四个频点间隔远大于 50 Hz 的 FFT 分辨率，逐符号峰值检测已经足够，无需保留频谱截图；调制参数和解码代码比图片更精确。
