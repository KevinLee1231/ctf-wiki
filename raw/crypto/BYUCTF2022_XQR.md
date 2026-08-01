# BYUCTF 2022 - xqr

## 题目简述

附件 `xqr.png` 是一张由大量 27×27 小二维码块组成的大图。题名把 XOR 与 QR 组合在一起，源码也表明最终二维码被拆散到了多个等尺寸位图中。

## 解题过程

将大图按 27 像素步长切成小块，并对所有块逐像素异或。仓库 `solve.py` 的核心逻辑是：

```python
from PIL import Image, ImageChops

result = None
for y in range(0, image.height, 27):
    for x in range(0, image.width, 27):
        tile = image.crop((x, y, x + 27, y + 27)).convert("1")
        result = tile if result is None else ImageChops.logical_xor(result, tile)
result.save("flag.png")
```

XOR 满足结合律与交换律，且随机遮罩出现偶数次会相互抵消，因此无需判断每个小块的用途。异或结果是一张可扫描二维码：

![全部小块逐像素异或得到的二维码](./BYUCTF2022_XQR/xor-result-qr.png)

扫码得到：

```text
byuctf{x0r_i5_u5eful}
```

## 方法总结

看到规则网格和题名中的 XOR，应优先验证“所有同尺寸块归约”而不是逐个扫码。处理二值图时要保持同一色彩模式和尺寸，否则图像库可能执行颜色混合而非逻辑异或。
