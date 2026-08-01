# GlacierCTF2022 - Size Matters

## 题目简述

附件是一段播放器难以正常处理的 WebM。视频帧会在播放过程中改变分辨率，flag 的 ASCII 码被编码在关键帧宽度中；容器层会妨碍工具正确报告逐帧尺寸，因此需要先提取 VP8 轨道。

## 解题过程

先从 Matroska/WebM 容器中取出视频轨，保存成 IVF，再让 libvpx 解码并记录尺寸变化：

```bash
mkvextract broken.webm tracks 0:video.ivf
ffmpeg -vcodec libvpx -i video.ivf frame-%03d.png 2>dimensions.log
```

基础尺寸为 $360\times360$。生成器对每个字符 $c$ 设置：

$$
w=360-\operatorname{ord}(c).
$$

视频还使用 bounce 效果改变高度，所以日志中会出现许多“宽度不变、只有高度变化”的记录；同一条 libvpx 消息也可能重复。解析 `dimension change! old_w x old_h -> new_w x new_h`，只在 `new_w` 与上一个宽度不同的时候取值：

```python
import re

current_width = 360
chars = []

for line in open("dimensions.log", encoding="utf-8", errors="ignore"):
    match = re.search(
        r"dimension change! \d+x\d+ -> (\d+)x\d+", line
    )
    if not match:
        continue
    new_width = int(match.group(1))
    if new_width != current_width:
        chars.append(chr(360 - new_width))
        current_width = new_width

print("".join(chars).strip("\0"))
```

首尾各有一个宽度 360 的 NUL 哨兵；去掉它们后得到：

```text
glacierctf{But_g1rth_15_m0r3_1mp0rt4nt}
```

## 方法总结

媒体文件的隐蔽信道不只存在于像素和音轨，编码参数本身也能传递数据。本题应把容器、编解码器日志和逐帧尺寸分层处理，并过滤 bounce 造成的高度噪声与重复日志；决定性障碍是从视频尺寸变化提取隐藏载荷，因此归 Stego。
