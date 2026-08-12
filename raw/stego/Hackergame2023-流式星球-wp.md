# 🪐 流式星球

## 题目简述

附件 `video.bin` 是一段没有容器、帧头和分辨率元数据的原始视频像素流，文件尾还被随机截去不超过 100 字节。目标是推断每帧的宽高，恢复能辨认的画面并从其中读取 flag。

决定性障碍不是视频解码器本身，而是从连续的三通道像素中重建隐藏的视觉载荷，因此归入 stego。

## 解题过程

### 明确字节布局

生成逻辑先用 OpenCV 逐帧读取视频，再把形状为 `(帧数, 高, 宽, 3)` 的 `uint8` 数组直接展平写入文件。OpenCV 返回的是 BGR 三通道，所以一个完整帧占用：

```text
width * height * 3 bytes
```

文件尾只损失至多 100 字节，前面的全部完整帧都不受影响。先按三个字节一像素观察，再调整候选行宽；当跨行错位消失、人物与文字稳定时，可确定宽为 427、高为 759。

### 提取最后一个完整帧

下面的脚本忽略尾部不足一个完整帧的数据，并把最后一个完整 BGR 帧写成 PNG。`cv2.imwrite` 接受 BGR 顺序，因此无需手动交换红蓝通道。

```python
from pathlib import Path

import cv2
import numpy as np

width, height = 427, 759
raw = np.frombuffer(Path("video.bin").read_bytes(), dtype=np.uint8)
frame_size = width * height * 3
frame_count = len(raw) // frame_size
assert frame_count > 0

last = raw[(frame_count - 1) * frame_size : frame_count * frame_size]
frame = last.reshape(height, width, 3)
assert cv2.imwrite("last-frame.png", frame)
```

恢复出的帧中可以直接读到 flag：

![按 427×759 重组原始像素流后恢复出的含 flag 视频帧](Hackergame2023-流式星球-wp/recovered-flag-frame.png)

帧中文字为：

```text
flag{it-could-be-easy-to-restore-video-with-haruhikage-even-without-metadata-0F7968CC}
```

文件总长度不能直接整除单帧大小是预期现象：这来自末尾的随机截断，而不是分辨率判断错误。

## 方法总结

这类裸媒体流应先从生成端的数据布局入手：确认通道数、通道顺序、行列次序和可能的截断位置，再用画面的周期性与连续性推断宽度。由于截断只发生在文件尾，选择最后一个完整帧即可绕开残帧，无需修复整个视频容器。
