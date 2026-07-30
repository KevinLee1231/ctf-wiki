# L3akCTF 2024 Communication Gateway Writeup

## 题目简述

附件 `file10.wav` 是一段 48 kHz、16 位、单声道音频，时长 22.4 秒。波形不是语音，而是在两种音调之间切换的低速 FSK 调制，并带有方波谐波。文件名中的 `10` 是有效提示：数据速率为 10 baud。

## 解题过程

先观察音频而不盲猜文本编码：

```console
ffprobe -v error \
  -show_entries format=duration:stream=sample_rate,channels,bits_per_sample \
  -of default=noprint_wrappers=1 file10.wav

sox file10.wav -n spectrogram -o gateway-spectrum.png
```

频谱显示稳定载波和按固定时间片切换的频率，符合 Bell-like FSK。`minimodem` 支持形如 `任意速率_N` 的自定义 Bell-like ASCII 模式，因此把文件名提示代入为 `10_N`：

```console
minimodem -f file10.wav 10_N
```

解调器锁定到约 1590 Hz 的载波，并输出：

```text
### CARRIER 10.00 @ 1590.0 Hz ###
L3AK{s1gn4ls_0f_h0p3}
### NOCARRIER ndata=22 confidence=33.694 ampl=0.636 bps=10.00 (rate perfect) ###
```

因此 flag 为：

```text
L3AK{s1gn4ls_0f_h0p3}
```

命令、载波参数和输出还可与[公开赛后题解](https://kashmir54.github.io/ctfs/L3akCTF2024/)交叉核对。外部记录的关键信息已经全部转写到正文，链接仅作为来源。

## 方法总结

这类题的关键不是“听起来像调制解调器”这一主观判断，而是三项可以验证的信号：离散载频、恒定符号宽度和文件名给出的 10 baud。`minimodem` 的 `10_N` 表示自定义 10 bps Bell-like ASCII 解调；若直接尝试常见的 Bell 202 1200 baud，符号率会差两个数量级而无法正确成帧。
