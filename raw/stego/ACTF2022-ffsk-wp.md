# FFSK

## 题目简述

题目只给出 `modem.wav` 和一段线索。“第二台商用计算机调制解调器”指向 Bell 103，“英国大城市”指向 Manchester 编码。音频本身是 48 kHz、单声道、16 位 PCM，共 17164800 个采样点；每 160 个采样构成一个 300 baud 符号，因此整段音频包含

$$
17164800/160=107280
$$

个符号。

生成源码表明，每个采样是两路 FSK 正弦波的叠加：应答端信道使用 2025/2225 Hz，发起端信道使用 1070/1270 Hz。前一信道承载解码提示，后一信道承载经过 Manchester 编码与 Hamming 纠错码包装的隐藏数据。

题目的决定性障碍是从音频的双路隐蔽信道中定位并逐层还原载荷，因此归入 Stego 合理；通信编码只是提取隐藏信息所需的具体手段。

## 解题过程

### 1. 分离两路 Bell 103 FSK

对每个 160 点窗口，分别计算两个候选频率的相关幅值。可使用 Goertzel 算法，也可直接计算单点 DFT：

$$
A_f(k)=\left|\sum_{n=0}^{159}x_{160k+n}e^{-j2\pi fn/48000}\right|.
$$

若 mark 频率的幅值更大，就记为 1；否则记为 0。分别使用下列频率即可同时解出两条 107280 位的原始比特流：

| 信道 | space / 0 | mark / 1 | 用途 |
| --- | ---: | ---: | --- |
| 应答端 | 2025 Hz | 2225 Hz | 明文提示 |
| 发起端 | 1070 Hz | 1270 Hz | 隐藏载荷 |

每个普通 Bell 103 字符帧由 10 位组成：1 位值为 0 的起始位、8 位低位在前的 ASCII 数据、1 位值为 1 的停止位。按这个格式解析 2025/2225 Hz 信道，会得到一段提示，其有效信息是：另一信道还经过 IEEE 802.3 风格的 Manchester 变换；使用 20 位 Hamming 块纠正人为加入的 bit flip；纠错后还要回到原始 Bell 103 字符帧。

仓库的 Goertzel 实现注明改写自 [laurenschneider/audiodecoder](https://github.com/laurenschneider/audiodecoder)。该项目同样以 Python 读取 WAV、按 Bell 103 频率解调，并特别处理了比特端序；这些信息已经在上文完整展开，链接仅保留为实现来源。

### 2. 逆向隐藏信道的编码层

1070/1270 Hz 信道的 107280 位不是直接的 UART 字符。生成器依次执行：

```text
Bell 103 字符帧 -> (20, 15) Hamming 编码并翻转一位 -> Manchester 编码 -> FSK
```

因此解码必须严格反向进行。

首先每两位解 Manchester。题目生成器采用 `1 -> 01`、`0 -> 10`，所以所有合法二元组只能是 `01` 或 `10`。107280 位解码后得到 53640 位。

然后每 20 位作为一个 Hamming 块。校验位位于从 1 开始计数的 $1,2,4,8,16$ 号位置，其余 15 位为数据。把块中所有值为 1 的位置异或，得到 syndrome：

$$
s=\bigoplus_{b_i=1}i.
$$

$s=0$ 表示无错；$1\le s\le20$ 时翻转第 $s$ 位。生成源码会在每个块中故意翻转且只翻转一位，因此所有 2682 个块都应得到范围内的非零 syndrome，这是判断同步与端序正确的重要依据。去掉 5 个校验位后，长度变为

$$
2682\times15=40230
$$

位。这里应以源码算出的 40230 为准；外部题解中偶见的 40320 是数字笔误。

最后把 40230 位按 10 位 Bell 103 帧解析，共得到 4023 个字符。文本以 `data:image/png;base64,` 开头，逗号后的内容是 PNG 的 Base64 数据。

### 3. 完整解码与结果

下面的脚本按上述顺序完成双路解调、Manchester 解码、Hamming 单错纠正、Bell 103 字符恢复和 PNG 导出：

```python
import base64
import sys
import wave
from pathlib import Path

import numpy as np


SAMPLES_PER_SYMBOL = 160


def read_pcm16_mono(path):
    with wave.open(str(path), "rb") as wav:
        if wav.getnchannels() != 1:
            raise ValueError("expected a mono WAV file")
        if wav.getsampwidth() != 2:
            raise ValueError("expected 16-bit PCM")
        rate = wav.getframerate()
        raw = wav.readframes(wav.getnframes())
    samples = np.frombuffer(raw, dtype="<i2").astype(np.float32)
    return rate, samples


def demodulate(samples, rate, mark_frequency, space_frequency):
    count = len(samples) // SAMPLES_PER_SYMBOL
    blocks = samples[:count * SAMPLES_PER_SYMBOL].reshape(
        count, SAMPLES_PER_SYMBOL
    )
    positions = np.arange(SAMPLES_PER_SYMBOL, dtype=np.float32)

    def basis(frequency):
        phase = -2j * np.pi * frequency * positions / rate
        return np.exp(phase).astype(np.complex64)

    mark_amplitude = np.abs(blocks @ basis(mark_frequency))
    space_amplitude = np.abs(blocks @ basis(space_frequency))
    return "".join(np.where(mark_amplitude > space_amplitude, "1", "0"))


def decode_bell103_frames(bits, stop_on_invalid=False):
    output = bytearray()
    for offset in range(0, len(bits) - 9, 10):
        frame = bits[offset:offset + 10]
        if frame[0] != "0" or frame[9] != "1":
            if stop_on_invalid:
                break
            raise ValueError(f"invalid Bell 103 frame at bit {offset}")
        value = sum(int(bit) << index for index, bit in enumerate(frame[1:9]))
        output.append(value)
    return bytes(output)


def decode_manchester(bits):
    if len(bits) % 2:
        raise ValueError("Manchester stream has odd length")
    decoded = []
    for offset in range(0, len(bits), 2):
        pair = bits[offset:offset + 2]
        if pair == "01":
            decoded.append("1")
        elif pair == "10":
            decoded.append("0")
        else:
            raise ValueError(f"invalid Manchester pair at bit {offset}: {pair}")
    return "".join(decoded)


def correct_hamming_block(block):
    if len(block) != 20:
        raise ValueError("Hamming block must contain 20 bits")
    code = int(block, 2)
    syndrome = 0
    for position in range(1, 21):
        if code & (1 << (position - 1)):
            syndrome ^= position
    if syndrome > 20:
        raise ValueError(f"invalid syndrome: {syndrome}")
    if syndrome:
        code ^= 1 << (syndrome - 1)

    data_low_to_high = []
    for position in range(1, 21):
        if position & (position - 1):
            bit = "1" if code & (1 << (position - 1)) else "0"
            data_low_to_high.append(bit)
    return "".join(data_low_to_high)[::-1], syndrome


def decode_hamming(bits):
    if len(bits) % 20:
        raise ValueError("encoded stream is not aligned to 20-bit blocks")
    output = []
    syndromes = []
    for offset in range(0, len(bits), 20):
        data, syndrome = correct_hamming_block(bits[offset:offset + 20])
        output.append(data)
        syndromes.append(syndrome)
    if any(not 1 <= syndrome <= 20 for syndrome in syndromes):
        raise ValueError("the challenge expects exactly one error in every block")
    return "".join(output)


def main():
    if len(sys.argv) != 2:
        raise SystemExit(f"usage: {sys.argv[0]} modem.wav")

    rate, samples = read_pcm16_mono(Path(sys.argv[1]))
    if rate != 48000:
        raise ValueError(f"unexpected sample rate: {rate}")

    answer_bits = demodulate(samples, rate, 2225, 2025)
    hint = decode_bell103_frames(answer_bits, stop_on_invalid=True)
    print(hint.decode("ascii"))

    originator_bits = demodulate(samples, rate, 1270, 1070)
    hamming_bits = decode_manchester(originator_bits)
    framed_bits = decode_hamming(hamming_bits)
    message = decode_bell103_frames(framed_bits).decode("ascii")

    prefix = "data:image/png;base64,"
    start = message.find(prefix)
    if start < 0:
        raise ValueError("PNG data URI was not found")
    encoded = message[start + len(prefix):].strip()
    Path("ffsk_qr.png").write_bytes(base64.b64decode(encoded, validate=True))
    print("wrote ffsk_qr.png")


if __name__ == "__main__":
    main()
```

扫描导出的二维码可得：

```text
ACTF{wow_h0w_IEEE_U_r}
```

[Maple Bacon 的参赛解法](https://maplebacon.org/2022/06/actf-ffsk/) 可作为独立验证：它先用 `minimodem --rx -M 2225 -S 2025 --file modem.wav 300` 读出提示，再从 17164800 个采样中按 160 点切片提取另一信道，最后经过同样的 Manchester、Hamming 和低位优先字符解码得到相同二维码。链接保留为替代实现，而不是正文成立的前置条件。

## 方法总结

完整数据链为“双路 Bell 103 FSK 叠加 → 1070/1270 Hz 信道的 Manchester 解码 → 20 位 Hamming 单错纠正 → 10 位低位优先 Bell 103 字符帧 → Base64 PNG → QR”。每层都有可核验的不变量：四个频率峰、每符号 160 个采样、Manchester 只出现 `01/10`、每个 Hamming 块恰有一个范围内 syndrome、最终字符流以 PNG data URI 开头。

这类信号隐写题应先根据采样率和频谱建立物理层参数，再逐层验证长度与编码约束。不要让工具的乱码输出代替判断：服务端信道给的是下一层规则，客户端信道才是载荷；端序、纠错和字符帧都必须按生成顺序的逆序处理。
