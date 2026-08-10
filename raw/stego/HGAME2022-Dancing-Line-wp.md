# Dancing Line

## 题目简述

附件是一张像素级折线路线图。路线只会向右或向下移动，两个黑色分隔块之间恰好包含 8 步；将向右记为 0、向下记为 1，每 8 位就能组成一个 ASCII 字符。决定性信息藏在图像的空间路径中，因此归入 Stego，而不是沿用官方 Crypto 外壳分类。

## 解题过程

从左上角的黑块出发，沿非白色像素行走。每个字符都读取 8 次方向：向右把当前值左移后加入 0，向下则加入 1。下面的脚本直接按路线遍历，不需要对整张图做 OCR：

```python
from PIL import Image

image = Image.open("Dancing Line.bmp").convert("RGB")
width, height = image.size
white = (255, 255, 255)

x = 0
y = 0
result = bytearray()

while True:
    value = 0
    for _ in range(8):
        if x + 1 < width and image.getpixel((x + 1, y)) != white:
            x += 1
            bit = 0
        elif y + 1 < height and image.getpixel((x, y + 1)) != white:
            y += 1
            bit = 1
        else:
            print(result.decode())
            raise SystemExit

        value = (value << 1) | bit

    result.append(value)
```

也可以先用已知前缀验证映射：字符 `h` 的 ASCII 二进制是 `01101000`（按 8 位补零），若第一段路线与之对应，就说明横纵方向没有写反。

最终输出为：

```text
hgame{Danc1ng_L1ne_15_fun,_15n't_1t?}
```

原 PDF 只保留了解码脚本而没有嵌入原始路线图；路线规则和最终输出通过 [YuGao 的 Dancing Line 复现](https://sxyugao.top/p/d379320f) 交叉核对。正文已经完整保留方向映射、分组长度和可执行算法。

## 方法总结

图像题首先要观察几何约束：路径只朝两个方向移动，使每一步天然对应一个二进制位；固定的 8 步间隔则对应 ASCII 字节边界。利用已知 flag 前缀验证位序后，沿线遍历比扫描整张像素矩阵更直接，也更不容易把背景或分隔块误判为数据。
