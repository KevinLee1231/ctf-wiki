# QRazy CSV

## 题目简述

CSV 的每一行保存一个黑色像素坐标。按照坐标重建 100×100 的二值图，就会得到可扫描的二维码。

## 解题过程

跳过表头，解析第一列中的 `x,y`，在白色画布上把对应位置设为黑色：

```python
import csv
from PIL import Image

image = Image.new("1", (100, 100), 1)
with open("secret_1 (1).csv", newline="") as fp:
    rows = csv.reader(fp)
    next(rows)
    for row in rows:
        x, y = map(int, row[0].split(","))
        image.putpixel((x, y), 0)

image.save("recreated-qr.png")
```

扫描重建出的二维码，得到：

```text
n00bz{qr_c0d3_1n_4_csv_f1l3_w0w!!!}
```

## 方法总结

CSV 不是普通数据表，而是图像的稀疏坐标表示。重建时要确认坐标顺序、画布尺寸和黑白值；二维码只是可再生的中间产物，无需作为 WP 资源长期保留。
