# Chiaroscuro

## 题目简述

附件是一段 16 位 PCM WAV 钢琴音频 `painted_audio.wav`。听感、频谱和常见 LSB 检查都没有明显异常，但 WAV comment 写着法语提示：`Le clair se décale vers l'obscur. Seul le prélude fut repeint.`。其中 *décalage de phase* 指向相位偏移，最后一句说明仅修改了开头一个 chunk。因此主障碍是音频相位隐写，分类为 stego，而不是普通音频取证。

编码器只改写首个 FFT chunk 的相位：在 Nyquist bin 左侧连续的一段频率 bin 中，`+π/2` 表示比特 `0`，`-π/2` 表示比特 `1`；共轭位置写入反号、逆序相位，使 IFFT 仍是实数音频。其余采样保持不变，所以幅度频谱和听感都不是可靠入口。

## 解题过程

### 定位嵌入区

先检查元数据而不是反复尝试音频 LSB：

```bash
exiftool painted_audio.wav
```

枚举小的 2 的幂 chunk。对首 chunk 做 FFT 后，从 `mid - 1` 向左找一段连续的相位 `±π/2`；必须从末端反向计算连续长度，因为自然音频中也可能零星出现接近 `±π/2` 的相位。

本题得到 `chunk_size = 512`、`msg_len = 224`。224 正好是 28 个 ASCII 字符的 224 bit，且位置为 `phases[32:256]`。

### 还原比特与文本

下面代码完整复现官方 decoder 的核心逻辑；它只读取原始 WAV，不会改写音频。

```python
import wave

import numpy as np


def find_layout(samples: np.ndarray) -> tuple[int, int]:
    for chunk_size in (128, 256, 512, 1024, 2048, 4096):
        phases = np.angle(np.fft.fft(samples[:chunk_size]))
        mid = chunk_size // 2
        msg_len = 0
        for index in range(mid - 1, -1, -1):
            if abs(abs(phases[index]) - np.pi / 2) < 0.05:
                msg_len += 1
            else:
                break
        if msg_len >= 6 * 8:
            return chunk_size, msg_len
    raise ValueError("未找到连续的相位编码区")


with wave.open("painted_audio.wav", "rb") as wav:
    samples = np.frombuffer(wav.readframes(wav.getnframes()), dtype=np.int16)

chunk_size, msg_len = find_layout(samples)
phases = np.angle(np.fft.fft(samples[:chunk_size]))
mid = chunk_size // 2
bits = "".join("1" if phase < 0 else "0" for phase in phases[mid - msg_len:mid])
text = "".join(chr(int(bits[i:i + 8], 2)) for i in range(0, msg_len, 8))
print(chunk_size, msg_len, text)
```

输出为：

```text
512 224 grey{p41n73d_47_p1_0v3r_7w0}
```

## 方法总结

- 核心技巧：相位编码可以在不显著改变幅度谱和听感的情况下存放消息；解码时直接观察 FFT phase。
- 识别信号：音频题的元数据出现 *shift*、*phase*、*décalage* 等词，且频谱/LSB 均无异常时，应检查相位而非继续搜索可见频率图案。
- 复用要点：实数信号的频谱必须保持共轭对称；反向寻找连续的 `±π/2` 尾段能降低噪声 bin 对消息长度判断的干扰。
