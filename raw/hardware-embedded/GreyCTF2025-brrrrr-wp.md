# GreyCTF 2025 brrrrrrrrr Writeup

## 题目简述

附件是一段听起来持续“嗡嗡”作响的 FLAC 音频。其内容不是语音或频谱图，而是用 $1\ \text{kHz}$ 正弦载波承载的二进制相移键控信号。每个数据位占 48 个采样点，采样率为 $48\ \text{kHz}$，因此符号率为 $1000$ bit/s。

解题目标是去掉头尾填充，做相干解调并按八位一组还原 ASCII 文本。由于决定性步骤是对采样通信信号进行 PSK 解调，归入 hardware-embedded。

## 解题过程

先将 FLAC 解码为单声道采样数组。生成端在真实信号前添加 1234 个采样点、末尾添加 4321 个采样点，所以先切片：

```python
signal = signal[1234:-4321]
```

生成端把字符转成八位二进制，将 `1` 映射为 $+1$、`0` 映射为 $-1$，每位重复 48 次，再乘以载波：

$$
s[n] = a_k\sin\left(2\pi\cdot1000\frac{n}{48000}\right),
\qquad a_k\in\{-1,+1\}.
$$

因此可生成同频、同相的本地正弦载波，与采样逐点相乘。相位相同的符号得到正能量，相位翻转的符号得到负能量；每 48 点积分一次就能完成判决：

```python
t = np.arange(len(signal)) / 48000
carrier = np.sin(2 * np.pi * 1000 * t)
mixed = signal * carrier
symbols = mixed.reshape(-1, 48).sum(axis=1)
bits = (symbols > 60000).astype(int)
```

把判决结果连接为比特串，每八位转成一个字符：

```python
bit_string = ''.join(map(str, bits))
text = ''.join(
    chr(int(bit_string[i:i + 8], 2))
    for i in range(0, len(bit_string), 8)
)
```

生成端把同一明文重复了 20 次，解调结果也会连续出现 20 份相同字符串。取其中一份即可：

```text
grey{ph4s3_sh1ft_k3y1ng_3498th45789tgh}
```

## 方法总结

本题利用 BPSK 的 $0/\pi$ 相位差编码二进制。载波频率、采样率和每符号采样数满足整数周期关系，所以无需复杂的时钟恢复；裁掉固定填充后，用同相载波混频并对每个符号窗口求和即可。明文重复 20 次既增强了可识别性，也可用于检查同步是否正确：若每段输出一致，说明切片位置、相位和位宽均已对齐。
