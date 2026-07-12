# d3gif

## 题目简述

题目把二维码像素信息藏在 GIF 帧的 RGB 值中。文件名提示 `(x,y,bin).gif`：每一帧左上角像素的 RGB 三元组分别表示二维码坐标 `x`、`y` 和黑白位 `bin`。遍历所有帧后按坐标重建图片即可得到 QR code。

## 解题过程

文件名为 `(x,y,bin).gif`，其中 `x, y, bin` 对应每一帧的 RGB 值。`x` 和 `y` 是坐标，`bin` 是二值位，用来表示 QRCode 中的黑白像素。

```
from PIL import Image
img = Image.open('(x,y,bin).gif')
coorInfo = []
x_max = 0
y_max = 0
try:
    frame = 0
    while True:
        img.seek(frame)
        rgb = img.convert("RGB").getpixel((0, 0))
        if rgb == 0:
            rgb = (0, 0, 0)
        coorInfo.append(rgb)
        x_max = max(x_max, rgb[0])
        y_max = max(y_max, rgb[1])
        frame += 1
except EOFError:
    pass
img = Image.new('RGB', (x_max + 1, y_max + 1))
for x, y, bin in coorInfo:
    img.putpixel((x, y), (255,255,255) if bin else (0,0,0))
img.save('flag.png')
```

## 方法总结

- 核心技巧：GIF 多帧遍历、用 RGB 通道编码坐标和像素值、重建 QR code。
- 识别信号：文件名或题面直接给出 `(x,y,bit)` 结构时，优先检查每帧固定位置像素，而不是对整张 GIF 做常规隐写扫描。
- 复用要点：先统计最大 `x/y` 决定输出图片尺寸，再逐帧 `seek` 读取 RGB，按 `bin` 写黑白像素。
