# maplewave-0

## 题目简述

题目给出录音程序和 `flag0.maplewave`，说明音频采样率为 16 kHz，听到的英文单词用空格连接，末尾是数字。文件不是现成 WAV，但开头有自定义魔数 `MPLEWAVE`。`maplewave-0` 使用最简单的 codec 0，重点是恢复容器头和原始采样格式。

## 解题过程

文件前 16 字节结构如下：

```text
0x00  8 bytes  "MPLEWAVE"
0x08  1 byte   codec
0x09  3 bytes  padding
0x0c  4 bytes  frame count
0x10  ...      encoded samples
```

codec 0 不做压缩，`0x10` 之后就是单声道 unsigned 8-bit PCM。可以直接构造 WAV 头，或用 Python 写出采样：

```python
import wave

payload = open("flag0.maplewave", "rb").read()[16:]
with wave.open("flag0.wav", "wb") as out:
    out.setnchannels(1)
    out.setsampwidth(1)
    out.setframerate(16000)
    out.writeframes(payload)
```

播放后语音为 “easy pulse code modulation”，随后逐个读出数字 5、7、1、6。按题目正则包装：

```text
maple{easy pulse code modulation 5716}
```

## 方法总结

自定义音频容器首先要区分“容器元数据”和“采样数据”。通过录音程序调用的 PulseAudio 参数或源码可确认 1 通道、16 kHz、unsigned 8-bit；若误用常见的 16-bit PCM，播放只会得到噪声。语音中的末尾数字应逐位记录，并用题目正则复核空格和数字格式。
