# Scout out the Flag

## 题目简述

附件是一张几乎全白的 PNG。图像结构正常，也没有额外尾随数据；直接观察时只能隐约看到一行比白色背景稍暗的文字，属于低对比度视觉隐藏。

## 解题过程

原图中的有效信息如下。文字像素与背景的亮度差很小，但放大后已经能辨认其轮廓：

![近乎纯白的画布中央藏有一行极低对比度的浅灰色 flag 文字](UMDCTF2017-scout-out-the-flag-wp/faint-flag.png)

使用图像编辑器的色阶或自动对比度，把实际像素最小值映射到黑色、最大值映射到白色，即可清晰显示文本。Pillow 的等价处理为：

```python
from PIL import Image, ImageOps

image = Image.open("easy_peasy_steg.png").convert("L")
revealed = ImageOps.autocontrast(image)
revealed.save("revealed.png")
```

读取得到：

```text
UMDCTF-{easy_peasy_right?}
```

该字符串的 SHA-256 与 README 中的 `4d85e0cabd615e04afa494804fdb60f347a2d340743074391b05be772a7ef64c` 一致。

## 方法总结

“看起来全白”不等于像素完全相同。先查看直方图、最小值和最大值，再拉伸动态范围，是处理低对比度隐写的基本步骤。本题的 flag 就在正常像素层中，不需要 LSB 或文件尾提取。
