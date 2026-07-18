# week2differentPic

## 题目简述

题目给出两张同为 `500 × 350` 的 PNG。肉眼观察内容接近，但两图像素之间的细微差异承载了一张二维码。直接看普通差分会混入大量噪声；在 StegSolve 的 Image Combiner 中执行 SUB 后，二维码主要出现在蓝色通道最低位 `blue plane 0`。

## 解题过程

在 StegSolve 中分别载入 `lena.png` 和 `flag.png`，进入 Image Combiner，切换到 SUB 模式。把差分结果另存并重新打开，选择 `blue plane 0`，即可看到二维码轮廓。二维码周围仍有细线噪声，缩小显示或使用远距离扫描通常更容易识别。

也可以直接计算两图蓝色通道最低位的异或。减法结果的最低位与异或最低位相同，因此无需真的保存中间 SUB 图：

```python
from PIL import Image

left = Image.open("lena.png").convert("RGB")
right = Image.open("flag.png").convert("RGB")
if left.size != right.size:
    raise ValueError("两张图片尺寸不一致")

plane = Image.new("1", left.size, 1)
left_pixels = left.load()
right_pixels = right.load()
out = plane.load()

for y in range(left.height):
    for x in range(left.width):
        left_blue = left_pixels[x, y][2]
        right_blue = right_pixels[x, y][2]
        bit = (left_blue ^ right_blue) & 1
        out[x, y] = 0 if bit else 1

plane.save("difference-blue-lsb.png")
```

打开 `difference-blue-lsb.png`，聚焦中间的三个定位符区域并扫描二维码，即可得到 flag。保存或放大时必须使用最近邻插值，不能让黑白模块被平滑成灰色。

## 方法总结

- 核心技巧：对齐两张图做像素差分，再查看差分结果的蓝通道最低位。
- 识别信号：两图外观几乎一致，但逐像素值存在细小变化；差分位面中出现二维码定位符。
- 复用要点：先保证尺寸和坐标完全一致；低位图应二值化并用最近邻缩放，噪声较多时优先识别定位符而不是全图阈值。
