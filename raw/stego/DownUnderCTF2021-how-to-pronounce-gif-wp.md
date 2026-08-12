# DownUnderCTF 2021 - How to pronounce GIF?

## 题目简述

附件是一个 120 帧、每帧仅 $300\times22$ 像素的快速 GIF。单帧只显示 QR 图的一条水平带；实际有 10 张 QR 被按时间交错，每张 QR 由相隔 10 帧的 12 条带自上而下组成。决定性步骤是按帧索引重组空间结构，而不是读取某一帧的文字。

![120 帧 GIF 中快速交错播放的十组 QR 水平条带](DownUnderCTF2021-how-to-pronounce-gif-wp/interleaved-qr-strips.gif)

## 解题过程

将 GIF 无损拆帧并保持原始顺序。若 QR 编号为 $j\in[0,9]$，它的条带索引为：

$$
j,\ j+10,\ j+20,\ldots,\ j+110.
$$

下面的 Pillow 脚本按该规则纵向拼接，每张结果图大小为 $300\times264$：

```python
from PIL import Image

gif = Image.open("interleaved-qr-strips.gif")
frames = []

for index in range(gif.n_frames):
    gif.seek(index)
    frames.append(gif.convert("RGB").copy())

if len(frames) != 120:
    raise ValueError(f"unexpected frame count: {len(frames)}")

for qr_index in range(10):
    strips = [frames[qr_index + 10 * row] for row in range(12)]
    width = strips[0].width
    height = sum(strip.height for strip in strips)
    canvas = Image.new("RGB", (width, height), "white")

    y = 0
    for strip in strips:
        canvas.paste(strip, (0, y))
        y += strip.height

    canvas.save(f"qr-{qr_index:02d}.png")
```

用 `zbarimg qr-*.png` 或任意离线 QR 解码器按编号读取。10 张 QR 中多数是提示或干扰项；与 flag 直接相关的是第 7、9 组，它们分别给出 Base64 的前后两段：

```text
RFVDVEZ7YU1
fMV9oYVhYMHJfbjB3P30=
```

按时间顺序连接后解码：

```bash
printf '%s' 'RFVDVEZ7YU1fMV9oYVhYMHJfbjB3P30=' | base64 -d
```

得到：

```text
DUCTF{aM_1_haXX0r_n0w?}
```

## 方法总结

GIF 每帧高度异常小、播放极快，说明信息被拆成条带并沿时间轴交错。处理帧隐写时要保留帧顺序、调色板转换和 disposal 后的实际像素，先观察尺寸与帧数的因数关系；本题的 120 帧正好分成 10 组、每组 12 条。重组后再做 QR 与 Base64 解码，能够清楚区分“隐藏载荷定位”与后续普通编码步骤。
