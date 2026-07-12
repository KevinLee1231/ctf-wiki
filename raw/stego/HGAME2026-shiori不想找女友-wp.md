# Shiori不想找女友

## 题目简述

题面要求根据 Shiori 留下的头像寻找位置。附件实际只有一张图片 `shiori.png`；图像本身有明显噪点，EXIF 中藏有一段 JSON 参数，描述从图片中抽样读取信息所需的块大小、起始坐标、横纵步长和列数。当前目录里没有额外压缩包，后续内容都从这张图片的像素/元数据中提取。

## 解题过程

附件只有一张 `shiori.png`。图片上有明显的规则噪点，先查看 EXIF 信息，可以发现里面放了一段 JSON 参数：

```
{"block": 1, "start_x": 10, "start_y": 10, "step_x": 7, "step_y": 7,
"column_num": 450}
```

这些参数描述了如何从原图中抽样恢复隐藏图像：`block` 是块大小，`start_x/start_y` 是起始坐标，`step_x/step_y` 是噪点分布间隔，`column_num` 是恢复后图像的列数。写脚本按网格取块并重排即可复原隐藏内容：

```python
from PIL import Image
CHALLENGE_FILE = "shiori.png"
OUTPUT_FILE = "key.png"
def solve():
    img = Image.open(CHALLENGE_FILE)
    b = 1
    sx = 10
    sy = 10
    dx = 7
    dy = 7
    w_blocks = 450
    canvas_w = w_blocks * b
    canvas_h = 600
    restored_img = Image.new('RGB', (canvas_w, canvas_h), 'lightgray')
    c_w, c_h = img.size
    carrier_x, carrier_y = sx, sy
    target_col, target_row = 0, 0
    count = 0
    while True:
        if carrier_y + b > c_h:
            break
        box = (carrier_x, carrier_y, carrier_x + b, carrier_y + b)
        block = img.crop(box)
        paste_x = target_col * b
        paste_y = target_row * b
        if paste_y + b > canvas_h:
            break
        restored_img.paste(block, (paste_x, paste_y))
        count += 1
        target_col += 1
        if target_col >= w_blocks:
            target_col = 0
            target_row += 1
        carrier_x += dx
        if carrier_x + b > c_w:
            carrier_x = sx
            carrier_y += dy
    restored_img.save(OUTPUT_FILE)
if __name__ == "__main__":
    solve()
```

恢复出的图像中可读出：

```text
This_is_a_key_for_u
```

当前本地附件只有这张图片，因此 WP 中不再写“加密压缩包”作为题目必要条件。能从附件稳定复现的关键过程是：从 EXIF 读取采样参数，按网格抽样恢复隐藏文本线索。

## 方法总结

图片隐写题要先查 EXIF/元数据，再看图像中规则噪点是否对应坐标采样。题目给出 start/step/column 这类参数时，应直接写脚本按网格取像素；恢复出的图像、JWT、base64 或 JSON 再作为下一阶段线索处理。
