# Sum of Sines

## 题目简述

附件 `matlab.mat` 中只有一个长度为 1000 的浮点信号 `signal`。题目提到 “ASCII-betical”、sum of sines 和 “transforming”，说明每个字符被当作频率，多个正弦波再叠加到同一信号中，需要用傅里叶变换恢复频率集合。

## 解题过程

对信号做实数 FFT，观察正频率幅值。明显峰值位于：

```text
66, 69, 70, 89, 95, 97, 101, 103, 105, 108, 111, 112, 115
```

其中频点 69 的幅值是其他峰值的两倍，说明字符 `E` 出现了两次。每个基波的幅值约为 500，可以按幅值除以 500 恢复重复次数：

```python
import numpy as np
from scipy.io import loadmat

signal = loadmat("matlab.mat")["signal"].ravel()
spectrum = np.abs(np.fft.rfft(signal))

chars = []
for frequency, amplitude in enumerate(spectrum):
    count = round(amplitude / 500)
    if count:
        chars.extend(chr(frequency) for _ in range(count))

text = "".join(chars)
print(text)
```

频率本身就是零基 ASCII 码，而且题目说明字符串按 ASCII 顺序排列，所以输出为：

```text
BEEFY_aegilops
```

加上 flag 外壳：

```text
UMDCTF-{BEEFY_aegilops}
```

其 SHA-256 与 README 中的 `97b8f0263c9cb6024bc6fa32084463fe1c2845ac63dd3210691147cecade0c07` 一致。

## 方法总结

不同频率的正弦波在线性叠加后，时域中很难直接识别，但 FFT 会把能量重新集中到对应频点。除了峰的位置，还应利用幅值恢复重复字符；若在 MATLAB 中查看频谱索引，还要留意其数组从 1 开始，而题目编码使用零基频点。
