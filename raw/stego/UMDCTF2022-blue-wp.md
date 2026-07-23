# UMDCTF2022 Blue Writeup

## 题目简述

题目给出一张近似纯蓝色的 PNG 和生成脚本 `steg.py`。脚本没有把字符直接写进某个固定像素，而是对 flag 的第 $y$ 个字符执行 `ord(flag[y])` 次随机操作：每次在第 $y$ 行随机选择一个像素和一个 RGB 通道，将该通道加一。

因此，随机位置只是在分散改动；一整行相对原始底色的 RGB 增量总和仍然等于该字符的 ASCII 码。决定性机制是像素统计隐写，故归入 `stego`。

## 解题过程

生成脚本刻意不修改最后一列，所以右下角像素可以作为未被扰动的蓝色基准。设基准颜色为 $(R_0,G_0,B_0)$，第 $y$ 行像素为 $(R_{x,y},G_{x,y},B_{x,y})$，则该行编码的数值为：

$$
s_y=\sum_x\left((R_{x,y}-R_0)+(G_{x,y}-G_0)+(B_{x,y}-B_0)\right)
$$

随机选择像素和通道不会改变总增量，因此 `chr(s_y)` 就是对应字符。遇到总增量为零的第一行时停止：

```python
from PIL import Image

im = Image.open("bluer.png").convert("RGB")
width, height = im.size
base = im.getpixel((width - 1, height - 1))

flag = []
for y in range(height):
    value = 0
    for x in range(width):
        pixel = im.getpixel((x, y))
        value += sum(pixel[i] - base[i] for i in range(3))
    if value == 0:
        break
    flag.append(chr(value))

print("".join(flag))
```

得到：

```text
UMDCTF{L4rry_L0v3s_h3r_st3g0nogr@phy_89320}
```

## 方法总结

这类题不应只寻找最低有效位或固定通道。已知生成逻辑时，应先找在随机扰动下仍保持不变的统计量。本题的位置和通道都随机，但每次操作贡献的总增量恒为一，所以按行求 RGB 差值总和即可消除随机性。参考像素必须选在脚本明确不会改动的位置，否则会给整行引入系统偏差。
