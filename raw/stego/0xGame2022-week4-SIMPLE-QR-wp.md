# week4SIMPLE_QR

## 题目简述

题目有新、旧两个版本，载体都是经过破坏并夹带额外数据的二维码图片。新版本需要处理反色二维码、额外 zlib/IDAT 数据和一张缺少文件头的拼接 PNG；旧版本则需要直接解析一段 QR 数据码字。两条路线最终得到同一个 flag。

## 解题过程

### 新版本

先将附件反色，并补齐二维码定位图形与四周留白。扫描后得到前缀 `0xGame{ed4a6398-9360-`，其后还有 4 个字符尚需从另一张图恢复。

反色本身可直接完成，人工修复只需保证三个定位块仍是标准的黑白同心方框，且二维码外有空白静区：

~~~python
from PIL import Image, ImageOps

image = Image.open("SIMPLE_QR.png").convert("L")
ImageOps.invert(image).save("inverted.png")
~~~

继续检查 PNG 块结构，会发现正常图像数据之外还有一段可由 zlib 解压的额外 IDAT 数据。解压结果只含字符 `0` 和 `1`，长度为 $1089=33^2$，因此应按行排列成 $33\times33$ 的二维码。下面的脚本读取解压后的 `idat.bin`，按 `0` 为黑、`1` 为白绘制，并添加 4 模块静区和放大倍率以便扫码：

~~~python
from math import isqrt
from pathlib import Path

from PIL import Image

bits = Path("idat.bin").read_bytes().decode("ascii").strip()
side = isqrt(len(bits))
assert side == 33 and side * side == len(bits)
assert set(bits) <= {"0", "1"}

border = 4
scale = 10
qr = Image.new("L", ((side + 2 * border) * scale,) * 2, 255)

for index, bit in enumerate(bits):
    x = index % side + border
    y = index // side + border
    color = 0 if bit == "0" else 255
    for dy in range(scale):
        for dx in range(scale):
            qr.putpixel((x * scale + dx, y * scale + dy), color)

qr.save("idat-qr.png")
~~~

![将 1089 个 IDAT 比特按 33×33 排列后重建出的二维码](0xGame2022-week4-SIMPLE-QR-wp/idat-qr.jpg)

扫码得到末段：

~~~text
-9c89-82272f3c621e}
~~~

还缺中间 4 个字符。检查原文件可找到两个 `IEND` 块：第一张 PNG 完整结束后，又拼接了一张缺少开头 16 字节的 PNG。缺失部分正是 8 字节 PNG 签名，以及 4 字节 IHDR 数据长度和 4 字节块类型：

~~~text
89 50 4e 47 0d 0a 1a 0a 00 00 00 0d 49 48 44 52
~~~

从第一个 `IEND` 块及其 CRC 之后截取余下内容，补上这 16 字节即可恢复第二张图：

~~~python
from pathlib import Path

data = Path("SIMPLE_QR.png").read_bytes()
offset = 8
first_end = None
while offset + 12 <= len(data):
    chunk_length = int.from_bytes(data[offset:offset + 4], "big")
    chunk_type = data[offset + 4:offset + 8]
    chunk_end = offset + 12 + chunk_length
    if chunk_type == b"IEND":
        first_end = chunk_end
        break
    offset = chunk_end

if first_end is None:
    raise RuntimeError("没有找到第一个 IEND 块")

second_body = data[first_end:]
png_header_and_ihdr = bytes.fromhex(
    "89504e470d0a1a0a0000000d49484452"
)
Path("second.png").write_bytes(png_header_and_ihdr + second_body)
~~~

扫描 `second.png` 得到中间段 `41ff`，三段拼接后为：

~~~text
0xGame{ed4a6398-9360-41ff-9c89-82272f3c621e}
~~~

### 旧版本

旧版额外数据最终得到下面的十六进制串：

~~~text
4132d396338392d3832323732663363363231657d0ec386620386120366620306520343720636620
~~~

这段内容是 QR 的数据码字，而不是普通十六进制密文。其开头 4 位为模式标识；`0100` 表示字节模式。对于版本 1–9 的二维码，字节模式随后使用 8 位字符计数，本题读出 $19$；接下来的 $19\times8$ 位就是载荷。后续内容属于终止符、补齐位和剩余码字，不应继续当作正文解析。

~~~python
encoded = bytes.fromhex(
    "4132d396338392d3832323732663363363231657d0e"
    "c386620386120366620306520343720636620"
)
bits = "".join(f"{byte:08b}" for byte in encoded)

mode = bits[:4]
assert mode == "0100"

length = int(bits[4:12], 2)
assert length == 19

payload_bits = bits[12:12 + length * 8]
payload = int(payload_bits, 2).to_bytes(length, "big")
print(payload.decode())
~~~

输出仍是：

~~~text
-9c89-82272f3c621e}
~~~

结合另外两张二维码，同样得到完整 flag：

~~~text
0xGame{ed4a6398-9360-41ff-9c89-82272f3c621e}
~~~

## 方法总结

本题把信息分别藏在视觉层、PNG 容器层和 QR 码字层。检查二维码附件时，不应只尝试扫码，还要核对颜色、定位块、PNG 块序列、文件尾拼接和解压后数据尺寸；遇到原始 QR 码字，则按“模式标识 → 字符计数 → 定长载荷”的顺序解析，避免把终止与纠错相关数据误当成明文。
