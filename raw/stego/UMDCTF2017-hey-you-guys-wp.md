# Hey, you guys!

## 题目简述

`sloth.png` 可以正常显示，图像像素本身没有明显异常。检查 PNG 块结构后，会发现主图数据被人为拆成大量很短的 IDAT 块；这些块的长度不是压缩器通常产生的分段方式。

## 解题过程

PNG 块格式是：

```text
4 字节长度 | 4 字节类型 | 数据 | 4 字节 CRC
```

忽略正常的 IHDR、最后一个大 IDAT 和 IEND，把前面 26 个小 IDAT 的长度依次取出：

```text
85 77 68 67 84 70 45 123 83 49 48 84 72
95 76 48 86 69 53 95 67 72 85 78 75 125
```

这些数全部位于可打印 ASCII 范围。下面脚本直接解析块长度：

```python
import struct
from pathlib import Path

data = Path("sloth.png").read_bytes()
offset = 8
values = []

while offset < len(data):
    length = struct.unpack(">I", data[offset:offset + 4])[0]
    kind = data[offset + 4:offset + 8]
    if kind == b"IDAT" and length < 256:
        values.append(length)
    offset += 12 + length

print(bytes(values).decode())
```

输出：

```text
UMDCTF-{S10TH_L0VE5_CHUNK}
```

其 SHA-256 与 README 中的 `4ce05118c5eb8f0a989c54e31c2ea3bf75e0434e15ebf0189a7b3af509757427` 一致。

## 方法总结

PNG 隐写不仅可以利用像素、调色板和附加块，也可以利用“块长度”这种结构元数据。看到大量异常短、连续的同类型块时，应把长度、类型、CRC 或块顺序分别尝试为编码载体。本题无需解压 IDAT 内容。
