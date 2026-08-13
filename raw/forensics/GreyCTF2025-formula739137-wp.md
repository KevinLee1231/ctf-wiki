# Formula 739137

## 题目简述

题目给出一张 PNG，表面内容与 F1 新加坡大奖赛有关。真正载荷并不在像素，而是编码在 PNG 各 `IDAT` 块的 CRC 低字节中，因此归入数字取证。

## 解题过程

PNG 文件由 8 字节签名和一系列 chunk 构成；每个 chunk 依次包含大端长度、4 字节类型、数据和大端 CRC。逐块解析，在类型为 `IDAT` 时取 CRC 的最低 8 位：

```python
import struct

with open("challenge.png", "rb") as stream:
    stream.read(8)
    chars = []
    while True:
        length, chunk_type = struct.unpack(">I4s", stream.read(8))
        stream.read(length)
        crc, = struct.unpack(">I", stream.read(4))
        if chunk_type == b"IDAT":
            value = crc & 0xff
            if value in range(9, 14) or 32 <= value <= 126:
                chars.append(chr(value))
        if chunk_type == b"IEND":
            break

print("".join(chars))
```

CRC 值按文件顺序拼接后形成可打印文本，得到：

```text
grey{m3rced3s_1s_my_f4v_team!}
```

题图本身只是正常视觉载体，关键证据已经完整转为 chunk 结构和提取代码，因此无需额外保存截图。

## 方法总结

媒体取证不能只检查像素和常规元数据，还要验证容器结构本身。PNG 允许多个 IDAT 块，攻击者可调整数据使 CRC 的某些字节承载信息。解析时必须按大端格式跳过精确长度，并以 `IEND` 为终止；直接在文件中搜索可打印字符串通常看不到这种分散载荷。
