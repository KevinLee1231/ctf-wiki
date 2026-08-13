# subGB

## 题目简述

附件是一张仅 $60\times8$ 像素的 RGBA 图片，肉眼看起来像一条彩色噪声带。提示“把 RGB 画到网格上”说明信息并不由像素整体颜色表达，而是分别编码在每个像素的红、绿、蓝三个通道中。

![原始 60×8 RGBA 彩色像素带，每个像素的 RGB 三个通道共同承载隐藏点阵](./GreyCTF2025-subgb-wp/rgb-pixel-grid.png)

## 解题过程

每行包含 60 个像素。忽略 alpha，把每个像素的 `R`、`G`、`B` 顺序展开后，一行就从 60 个像素扩展为 180 个通道值。将值 `255` 画成 `*`，其他值画成空格，8 行通道数据会形成可读的点阵文字。

```python
from PIL import Image

image = Image.open("subGB.png").convert("RGBA")

for y in range(image.height):
    channels = []
    for x in range(image.width):
        channels.extend(image.getpixel((x, y))[:3])
    print("".join("*" if value == 255 else " " for value in channels))
```

也就是说，三个颜色通道并非组合成一个彩色符号，而是三个连续的黑白点。输出宽度为 $60\times3=180$，高度仍为 8；按原顺序阅读点阵即可得到：

```text
grey{Su6_P1x3m4L_m3s5A9iNg}
```

## 方法总结

面对尺寸异常小的图片，应同时检查像素维度、通道数和取值分布。本题若按普通 RGB 图像观看，三个独立比特位会被颜色混合掩盖；把通道轴展开为空间轴后，隐藏点阵才显现。alpha 只是无关的第四通道，必须排除，否则每个字符的横向比例会被破坏。
