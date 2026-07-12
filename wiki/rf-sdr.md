---
type: technique
tags: [hardware-embedded, forensics, technique, rf, sdr, iq, qam, fsk, signal]
skills: [ctf-hardware-embedded, ctf-forensics]
raw:
  - ../raw/hardware-embedded/rf-sdr.md
updated: 2026-06-18
---

# RF / SDR / IQ Signal Processing

## 适用场景

题目给 IQ dump、无线电采样、调制星座、频谱截图或 `.cf32/.cs16/.cu8` 原始信号，需要从物理层恢复符号、bitstream、帧结构和最终文本。这个页面属于 Hardware/Embedded；若只是普通媒体中的频谱隐藏文字，则转 Stego。

## 识别信号

- 文件名或题面出现 SDR、IQ、QAM、FSK、PSK、carrier、symbol rate、GNU Radio、RTL-SDR。
- 原始数据无法 `strings`，但按 complex samples 画频谱后有明显带宽。
- 星座图呈圆环、旋转、网格 blob 或多象限对称。
- payload 有 idle pattern、delimiter、preamble、frame length、CRC。

## 最小证据

- 已确认采样格式：complex64、int16 IQ、unsigned 8-bit IQ，且能画出频谱。
- 找到中心频率和大致 symbol rate，或者至少能把信号搬到 baseband。
- 星座/眼图/幅度时序中有可分簇结构，而不是纯噪声。

## 解法骨架

1. 用 `np.fromfile` 按候选 dtype 读取，先画 spectrum 和 waterfall。
2. 频移到 baseband，低通滤波，估计 symbol rate。
3. 根据星座判断调制：FSK/ASK/PSK/QAM；圆环通常是 carrier offset。
4. 做 carrier recovery、timing recovery、AGC，得到 symbol decisions。
5. 尝试 0/90/180/270 旋转、bit/nibble endian、delimiter framing 和 CRC。

## RF 解调分支

| 信号形态 | 处理重点 |
|---|---|
| cf32/cs16/cu8 | 先读对 dtype；读错格式会让频谱和星座全错。 |
| QAM-16 | 需要载波恢复和定时恢复，且可能有 4-fold rotation ambiguity。 |
| Cyclostationary | 符号率可从幅度平方频谱峰值估计。 |
| Framing | idle/sync/start/end delimiter 决定 bitstream 如何切分。 |

## 常见陷阱

- 直接 demod，不先确认 dtype、中心频率和采样率假设。
- 星座旋转时误判为噪声；圆环多是频偏。
- GNU Radio 默认 mapping 不一定是 Gray code，需要验证 constellation map。
- 解出 bitstream 后忘记尝试 endian、nibble order 和帧边界。

## 关联技巧

- [signals-and-hardware.md](signals-and-hardware.md)
- [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md)
- [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md)
- [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)
- [keyboard-mouse-audio-and-physical-puzzles.md](keyboard-mouse-audio-and-physical-puzzles.md)
- [forensics-tooling.md](forensics-tooling.md)

## 原始资料

- [rf-sdr.md](../raw/hardware-embedded/rf-sdr.md)
