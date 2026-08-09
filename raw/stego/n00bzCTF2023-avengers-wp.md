# Avengers

## 题目简述

附件是一段由大量二进制文本画面组成的视频。题面问“什么容纳了空间宝石”，答案 `Tesseract` 同时提示使用 Tesseract OCR；需要逐帧提取数字并按原顺序拼接。

## 解题过程

先用 OpenCV 顺序读取 `flag.avi` 的 114 帧，再对每帧做 OCR。官方脚本把构造器误写成了 `VideoFlag_video`，实际应使用 `cv2.VideoCapture`：

```python
import cv2
import pytesseract

video = cv2.VideoCapture("flag.avi")
bits = []
while True:
    ok, frame = video.read()
    if not ok:
        break
    text = pytesseract.image_to_string(frame, config="--psm 6")
    bits.extend(ch for ch in text if ch in "01")
video.release()

stream = "".join(bits)
flag = "".join(
    chr(int(stream[i:i + 8], 2))
    for i in range(0, len(stream), 8)
)
print(flag)
```

若个别帧识别不稳，可先转灰度、二值化并放大后重试。按 8 位一组转 ASCII，得到：

```text
n00bz{7h1s_1s_4_v3ry_l0ng_fl4g_s0_th4t_y0u_c4nn0t_s0lv3_7h3_ch4ll3ng3_m4nu4lly_b7w_73s3r4c7_1s_4_v3ry_g00d_t00l!}
```

## 方法总结

视频本身是隐藏载荷的时序容器，帧顺序不可打乱。OCR 后只保留 `0` 和 `1`，还应检查总位数能否被 8 整除，以便及时发现漏帧或误识别。
