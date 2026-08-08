# Only 4-Bit Depth?

## 题目简述

题目提供目标 `flag.bmp`，并开放一个已知明文 blackbox：向 `POST /api/render` 提交文本即可得到对应的 24-bit、无压缩 BMP。像素的颜色分量只会取 $8,24,40,\ldots,248$ 这 16 个离散值，每个通道实际承载一个 4-bit nibble，而不是正常的连续颜色。

编码器把 UTF-8 字节流的每个字节拆成高、低两个 nibble，依次写入 BMP 的原始 BGR 通道字节；正文后补零，并在 payload 最后附加 4 字节小端正文长度。解题目标是从 BMP 原始存储顺序恢复 nibble 流，再用尾部长度字段去掉填充。

## 解题过程

### 用已知明文确认映射

先分别提交 `A`、`AB`、`ABC` 或其它短字符串并下载结果：

```bash
curl -X POST -d 'text=A' '<challenge-host>/api/render' -o A.bmp
curl -X POST -d 'text=AB' '<challenge-host>/api/render' -o AB.bmp
```

观察像素原始通道值会发现它们总满足：

$$
c=16n+8,\qquad n\in\{0,1,\ldots,15\}.
$$

因此可由 $n=(c-8)/16$ 还原 nibble。例如字符 `a` 为 `0x61`，应产生 nibble `6`、`1`，对应通道值 `104`、`24`；其它单字符样本也符合这一规律。编码器每 3 个原始字节生成 6 个颜色分量，恰好占用两个 24-bit 像素。

### 正确处理 BMP 存储顺序

这里容易混淆“图像查看器显示的 RGB”和“文件中的原始字节”：

- 24-bit BMP 的像素字节按 B、G、R 顺序存储；
- 正高度 BMP 的显示坐标是自底向上，但本题编码器直接把 payload 的第一行写入文件的第一条像素行；
- 每行长度补齐到 4 字节边界，行尾 padding 不能当成颜色分量。

所以最稳妥的方法是直接解析 BMP 头，从像素偏移处按文件行顺序读取，每行只取 `width * 3` 个 BGR 字节，不经过会自动转为 RGB、自动翻转图像的图像库。如果从屏幕坐标读图，则等价顺序是从左到右、从底行到顶行，并在每个像素内按 B、G、R 取值。

### 完整恢复脚本

下面的脚本包含格式检查、行 padding、离散颜色验证、nibble 合并和长度截断：

```python
from pathlib import Path
import struct
import sys


def decode_bmp(path: str) -> str:
    data = Path(path).read_bytes()
    if len(data) < 54 or data[:2] != b"BM":
        raise ValueError("not a BMP file")

    pixel_offset = struct.unpack_from("<I", data, 10)[0]
    dib_size = struct.unpack_from("<I", data, 14)[0]
    width = struct.unpack_from("<i", data, 18)[0]
    height = struct.unpack_from("<i", data, 22)[0]
    planes = struct.unpack_from("<H", data, 26)[0]
    bit_count = struct.unpack_from("<H", data, 28)[0]
    compression = struct.unpack_from("<I", data, 30)[0]

    if dib_size < 40 or planes != 1 or bit_count != 24 or compression != 0:
        raise ValueError("expected an uncompressed 24-bit BMP")

    width, height = abs(width), abs(height)
    row_bytes = width * 3
    stride = (row_bytes + 3) & ~3
    pixel_size = stride * height
    if pixel_offset + pixel_size > len(data):
        raise ValueError("truncated pixel data")

    # 保留文件中的行顺序和 BGR 通道顺序，只去掉行尾 padding。
    channels = bytearray()
    for row in range(height):
        start = pixel_offset + row * stride
        channels.extend(data[start:start + row_bytes])

    nibbles = []
    for channel in channels:
        if channel < 8 or (channel - 8) % 16 != 0:
            raise ValueError(f"unexpected color component: {channel}")
        nibble = (channel - 8) // 16
        if not 0 <= nibble <= 15:
            raise ValueError(f"nibble out of range: {nibble}")
        nibbles.append(nibble)

    payload = bytes(
        (nibbles[i] << 4) | nibbles[i + 1]
        for i in range(0, len(nibbles), 2)
    )
    if len(payload) < 4:
        raise ValueError("payload too short")

    text_length = struct.unpack("<I", payload[-4:])[0]
    if text_length > len(payload) - 4:
        raise ValueError("invalid length field")
    return payload[:text_length].decode("utf-8")


if __name__ == "__main__":
    print(decode_bmp(sys.argv[1]))
```

对题目提供的 `flag.bmp` 运行：

```bash
python3 solve.py flag.bmp
```

当前源码的本地构建配置用同一编码器生成测试附件，解码结果为：

```text
miniL{7e815c3d-fcff-4386-91e6-1bdcb17efe8d}
```

解码器取到的长度字段、UTF-8 解码和 `miniL{...}` 格式共同构成验证；不能仅凭像素中偶然出现的可打印字符判定成功。

## 方法总结

- 核心技巧：把 16 档离散颜色还原为 4-bit nibble，按 BMP 原始 BGR 与文件行顺序拼回字节，再读取末尾小端长度字段去除填充。
- 识别信号：颜色通道集中在等间隔档位、服务支持已知明文到图片的转换、短输入只改变少数像素，都表明像素通道本身被当作定长符号使用。
- 复用要点：先用单字节样本确定高低 nibble 顺序，再确认通道顺序、行方向和 padding；读取文件原始像素通常比依赖图像库的 RGB 视图更不容易发生隐式重排。恢复后还要利用长度、编码和 flag 格式做多重校验。
