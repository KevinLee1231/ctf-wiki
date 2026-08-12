# p😭q

## 题目简述

附件是一张 587 帧的 GIF，每帧用 32 组红色柱状条表示一个时间片的量化梅尔频谱。题目明确花括号内是 12 位数字。生成过程先对 22050 Hz 音频做 $n\_fft=2048$、hop length 512、Hann 窗的短时傅里叶变换，再压缩为 32 个 mel 频带、转为 $[-60,30]$ dB 并以 2 dB 为步长量化。目标是解析柱高，近似逆变换为语音。

![随时间播放的 32 频带量化梅尔频谱动画](Hackergame2021-pq-wp/mel-spectrogram.gif)

## 解题过程

### 从 GIF 恢复 dB 矩阵

每个红色柱的高度编码一个 mel 频带的分贝值。按生成代码的采样位置读取蓝色通道，统计值为 0 的像素数，再乘量化步长并加最小值：

```python
import numpy as np
from PIL import Image, ImageSequence

NUM_FREQS = 32
QUANTIZE = 2
MIN_DB = -60

gif = Image.open("mel-spectrogram.gif")
frames = np.array([
    np.asarray(frame.copy().convert("RGB"), dtype=np.uint8)
    for frame in ImageSequence.Iterator(gif)
])

log_mel = np.array([
    [
        (frame[::QUANTIZE,
               band * (QUANTIZE * 2) + QUANTIZE,
               2] == 0).sum() * QUANTIZE + MIN_DB
        for band in range(NUM_FREQS)
    ]
    for frame in frames
], dtype=float).T
```

转置后 `log_mel` 的形状是 `(32, 587)`，即频带在前、时间帧在后。

### 近似恢复音频

频谱生成时已经丢失 STFT 相位，mel 滤波也是有损降维，因此不存在精确逆变换。但 [librosa 的 `mel_to_audio`](https://librosa.org/doc/main/generated/librosa.feature.inverse.mel_to_audio.html) 会先近似恢复线性频谱，再使用 Griffin-Lim 迭代估计相位；语音辨识所需的信息仍然足够。

```python
import librosa
import soundfile as sf

SR = 22050
N_FFT = 2048
HOP_LENGTH = 512

mel_power = librosa.db_to_power(log_mel)
audio = librosa.feature.inverse.mel_to_audio(
    mel_power,
    sr=SR,
    n_fft=N_FFT,
    hop_length=HOP_LENGTH,
    window="hann",
)
sf.write("recovered.wav", audio, SR)
```

播放恢复音频，可以听到逐位读出的数字：

```text
634971243582
```

因此 flag 为：

```text
flag{634971243582}
```

## 方法总结

- 核心技巧：从动画柱高恢复量化 log-mel 矩阵，用 `db_to_power` 和 Griffin-Lim 路线近似重建语音。
- 识别信号：频谱动画、固定数量频带、已给生成参数，以及题目提示输出仅由数字组成。
- 复用要点：相位与 mel 压缩造成的信息损失不妨碍语音可懂度；数组轴顺序、量化步长、dB 基准和窗函数必须与生成端一致。
