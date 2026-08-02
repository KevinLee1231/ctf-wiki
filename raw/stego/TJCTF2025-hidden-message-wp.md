# hidden-message

## 题目简述

附件是一张普通 RGB 渐变图。生成代码把 `flag + "###END###"` 转为逐字节大端比特流，按行优先遍历所有像素，并依次写入每个像素 R、G、B 三个通道的最低有效位。视觉上几乎看不出变化，决定性信息位于 RGB LSB 通道。

## 解题过程

严格复现嵌入顺序：像素按 Pillow 的行优先顺序读取，每个像素依次取 R、G、B 的最低位，每 8 位还原一个字符，遇到终止串立即停止。

```python
from PIL import Image

with Image.open("suspicious.png") as image:
    pixels = image.convert("RGB").getdata()

bits = []
for red, green, blue in pixels:
    bits.extend((red & 1, green & 1, blue & 1))

message = bytearray()
for start in range(0, len(bits) - 7, 8):
    value = 0
    for bit in bits[start:start + 8]:
        value = (value << 1) | bit
    message.append(value)
    if message.endswith(b"###END###"):
        del message[-9:]
        break

print(message.decode())
```

实际输出为：

```text
tjctf{steganography_is_fun}
```

原图只是渐变背景，肉眼无法观察到 LSB 差异；因此不保留一张不能表达隐藏通道的装饰性图片，正文中的通道顺序和终止条件更有复现价值。

## 方法总结

- 核心技巧：按 RGB、逐像素、逐通道顺序提取最低有效位并重组字节。
- 识别信号：无损 PNG、视觉正常但题目强调 hidden message、生成代码用 `& 0xFE | bit` 修改通道。
- 复用要点：必须确认通道顺序、扫描顺序、位序和终止条件；任一项错误都会产生看似随机的文本。
