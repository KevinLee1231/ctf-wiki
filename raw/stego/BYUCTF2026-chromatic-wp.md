# Chromatic

## 题目简述

附件是一段 54 秒、30 FPS 的 H.264 视频。画面看起来只是随时间变化的纯色，没有可辨认图形；flag 被编码在每秒画面的红色通道数值中。

## 解题过程

先用媒体信息确认帧率为 30 FPS、总帧数为 1620。每隔 30 帧取一帧，并读取画面中心像素。由于每帧颜色基本均匀，空间位置不重要；把红色通道值按 ASCII 转成字符即可：

    chars = []
    for frame_index in range(0, 1620, 30):
        frame = read_frame(frame_index)
        red = frame[frame.shape[0] // 2, frame.shape[1] // 2, 2]
        chars.append(chr(int(red)))
    print("".join(chars))

54 个采样值依次给出：

    byuctf{It's_all_red_I_really_thought_it_would_be_more}

视频帧没有图案、构图或其他视觉证据，保留任意截图只会重复一个 RGB 数值，因此不归档图片。

## 方法总结

面对纯色或近纯色视频，应把时间轴和颜色通道都视为数据载体。先用帧数与预期文本长度推断采样周期，再分别测试 RGB 通道并用 flag 前缀验证。读取库常用 BGR 顺序，红色通常位于索引 2，这是实现时最容易混淆的一点。
