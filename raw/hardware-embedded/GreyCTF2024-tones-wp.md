# GreyCTF2024 Tones WP

## 题目简述

附件 `flag.flac` 使用七进制 FSK 编码文本。采样率为 12000 Hz，每个符号持续 1200 个采样点；符号 $s\in\{0,\ldots,6\}$ 对应频率 $300+85s$ Hz。每个 ASCII 字符再表示成三个七进制数字。

## 解题过程

先观察频谱，可以看到能量只落在七条固定频率线上：

```text
300, 385, 470, 555, 640, 725, 810 Hz
```

开头和结尾各有一段随机长度的静音。定位第一个有明显能量的 1200 点块后，对每块做实数 FFT，取峰值频率并映射为最接近的符号：

```python
import numpy as np
from scipy.io import wavfile

fs, x = wavfile.read("flag.flac")
x = x.astype(float)

# 用滑动能量定位第一个完整的非静音符号，得到 start。
digits = []
for pos in range(start, len(x) - 1199, 1200):
    block = x[pos:pos + 1200]
    spec = np.abs(np.fft.rfft(block))
    freq = np.fft.rfftfreq(len(block), 1 / fs)[np.argmax(spec)]
    digits.append(round((freq - 300) / 85))

text = "".join(
    chr(49 * digits[i] + 7 * digits[i + 1] + digits[i + 2])
    for i in range(0, len(digits) - 2, 3)
)
print(text)
```

三个数字按高位到低位排列，所以字符值为 $49a+7b+c$。解码结果中的 flag 为：

```text
grey{why_th3_7fsk_fr3qu3ncy_sh1ft_0349jf0erjf9jdsgdfg}
```

## 方法总结

本题的两层表示必须分开处理：先把固定频率映射为 $0$ 到 $6$，再把每三个七进制符号还原成 ASCII。符号长度固定，FFT 峰值间隔又远大于频率分辨率，因此不需要复杂的通信同步算法，只需正确跳过首尾静音。
