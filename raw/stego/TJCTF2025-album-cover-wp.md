# album-cover

## 题目简述

题目给出一张 $441\times444$ 的灰度 PNG 和生成代码。生成器读取单声道 16 位 PCM 音频，把连续 $441\times444=195804$ 个采样值线性映射到 0 至 255，再按行重排成图片。表面看到的“专辑封面”实际上是音频波形的二维排列；恢复音频后，还需要在频谱图中读取被绘制的 flag。

## 解题过程

生成器把有符号 16 位采样 $s$ 近似映射为

$$p=\left\lfloor\frac{s+32767}{65535}\times255\right\rfloor.$$

逆向时把灰度值按相反比例恢复到 `int16`，按行展开，并以生成代码对应的 44100 Hz、单声道、2 字节采样宽度写回 WAV：

```python
import wave
import numpy as np
from PIL import Image

pixels = np.asarray(
    Image.open("albumcover.png").convert("L"),
    dtype=np.float32,
)
samples = (pixels / 255 * 65535 - 32768).astype(np.int16).ravel()

with wave.open("restored.wav", "wb") as wav:
    wav.setnchannels(1)
    wav.setsampwidth(2)
    wav.setframerate(44100)
    wav.writeframes(samples.tobytes())
```

随后用 Audacity 切换到 Spectrogram 视图，或用 SoX 生成频谱图：

```bash
sox restored.wav -n spectrogram -o spectrogram.png
```

频谱中的大写文字为：

```text
tjctf{THIS-EASTER-EGG-IS-PRETTY-COOL}
```

原 PNG 视觉上只呈现近似噪声的横纹，真正的信息位于按音频解释后的时频分布中，因此正文保留转换参数和频谱结果即可。

## 方法总结

- 核心技巧：识别“像素值其实是 PCM 采样”的跨媒体编码，再从音频频谱读取隐藏文字。
- 识别信号：图片尺寸乘积恰好等于音频帧数，生成代码出现 `int16`、采样率和二维 `reshape`。
- 复用要点：恢复时必须保留样本顺序、位宽、声道数和采样率；普通播放听不清时，应继续检查频谱而不是认定恢复失败。
