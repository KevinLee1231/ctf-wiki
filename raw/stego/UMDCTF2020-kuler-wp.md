# Kuler

## 题目简述

题目提供一个网站颜色数据文件，里面连续出现大量 `0x...` 数值。这些颜色值实际承载二维码像素：每个十六进制数保留低 12 位，依次拼成一行黑白位图。

![从网站颜色值逐位重组并校正横纵比例后得到的二维码](UMDCTF2020-kuler-wp/website-color-qr.png)

## 解题过程

`prod/site_colors` 用空行分隔原始图像行。每组有 18 个数，每个数展开为固定宽度 12 位，共得到 216 位；实际二维码宽度为 200，因此丢弃行尾填充：

```python
from pathlib import Path
import re
from PIL import Image

rows = []
groups = Path("site_colors").read_text().strip().split("\n\n")
for group in groups:
    values = re.findall(r"0x[0-9a-fA-F]+", group)
    assert len(values) == 18
    bits = "".join(f"{int(value, 16) & 0xfff:012b}" for value in values)
    rows.append([255 if bit == "0" else 0 for bit in bits[:200]])

image = Image.new("L", (200, len(rows)), 255)
image.putdata([pixel for row in rows for pixel in row])
image = image.resize((200, 200), Image.Resampling.NEAREST)
image.save("website-color-qr.png")
```

原始数据是 200×100，横向像素重复造成二维码被压扁；按最近邻方式校正为方形后扫码得到：

```text
UMDCTF-{s0-M@ny_pr3tty_c0lors}
```

## 方法总结

把颜色常量视作整数而不是视觉配色后，规律就变成固定宽度位流。重组二维码时要同时处理每行填充和像素纵横比，缩放必须使用最近邻，避免插值产生灰边。
