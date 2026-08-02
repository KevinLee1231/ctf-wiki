# neofeudalism

## 题目简述

附件是一张肉眼正常的历史人物图片。生成器把 `flag + 长文本` 按位写入每个像素红色通道的最低有效位，遍历顺序为先 $x$ 后 $y$，并在每个字节内从最低位到最高位写入。

## 解题过程

直接读取红色通道 LSB，并按生成器的位序每 8 位还原一个字节：

```python
from PIL import Image

image = Image.open("image.png").convert("RGB")
bits = []
for x in range(image.width):
    for y in range(image.height):
        bits.append(image.getpixel((x, y))[0] & 1)

decoded = bytearray()
for offset in range(0, len(bits) - 7, 8):
    value = sum(bits[offset + bit] << bit for bit in range(8))
    decoded.append(value)

text = decoded.decode("utf-8", errors="ignore")
print(text[:text.index("}") + 1])
```

等价的 `zsteg` 通道选择是：

```bash
zsteg b1,r,msb,yx image.png
```

数据开头即为：

```text
tjctf{feudalism_still_bad_ea31e43b}
```

原图与载荷图在视觉上没有可辨差异，关键信息完全位于红通道 LSB，因此不重复保留肉眼等价的载体截图。

## 方法总结

- 无明显视觉异常的无损 PNG 应优先检查各颜色通道的低位平面和常见遍历顺序。
- 位序、通道与坐标顺序必须与嵌入端一致；“提取出乱码”往往只是参数选错，而非不存在载荷。
- 用 flag 前缀和右花括号截断，可以把后续用于掩护的长文本与真正目标分开。
