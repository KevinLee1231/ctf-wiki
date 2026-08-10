# New_Type_Steganography

## 题目简述

题目提供一个在线图片隐写服务和一张含 flag 的目标图，但没有直接给出算法源码。通过对纯色图片做选择明文测试，可以确认服务按固定伪随机顺序选择像素，并在绿色通道的第 2 位写入消息比特。由于随机种子固定、同尺寸图片生成的像素序列不变，可以借助在线编码接口恢复位置序列，再对照原始封面提取 flag。

## 解题过程

### 用选择明文判断嵌入方式

先上传与目标图同尺寸的纯白图片，分别嵌入 `A`、`AA`、`AAA` 等文本，并比较输入与输出：

- 只有少量像素发生变化，排除文件尾附加等格式层隐写；
- 相同长度消息重复使用相同位置，说明像素序列由固定种子决定；
- 新增的异常像素数与消息二进制中 `1` 的数量相关；
- 变化只发生在绿色通道的 `1 << 2` 位。

对绿色通道值 $g$，写入操作可概括为：

$$
g_0=g\mathbin{\&}\sim(1\ll2),
\qquad
g_1=g\mathbin{|}(1\ll2).
$$

在纯白图上 $g=255$，写 `0` 后变为 `251`，写 `1` 后仍为 `255`。因此“全零消息”的输出会暴露所有已使用位置，而把某一位单独改成 `1` 后，恰好有一个位置从 `251` 集合中消失。

### 恢复伪随机像素序列

假设 flag 是 ASCII，则每个字节最高位均为 `0`，只需探测其余 7 位。对第 $i$ 个字节：

1. 提交 `i + 1` 个 `\x00`，记录所有绿色通道为 `251` 的位置；
2. 保持前 $i$ 个字节为零，把最后一个字节依次改成 `0x40, 0x20, ..., 0x01`；
3. 每次用基线位置集合减去探测集合，唯一差值就是这一比特对应的像素。

恢复序列并解码目标图的脚本如下。比赛服务已经下线，`ORACLE_URL` 需在原环境或本地复现服务中替换：

```python
from io import BytesIO

import numpy as np
import requests
from PIL import Image

ORACLE_URL = "http://challenge.example/upload"
WHITE_IMAGE = "white.png"
ORIGINAL_IMAGE = "original.png"
STEGO_IMAGE = "flag.png"
MAX_BYTES = 64


def oracle(text):
    with open(WHITE_IMAGE, "rb") as image_file:
        response = requests.post(
            ORACLE_URL,
            data={"text": text},
            files={"file": ("white.png", image_file, "image/png")},
            timeout=15,
        )
    response.raise_for_status()
    return np.array(Image.open(BytesIO(response.content)).convert("RGB"))


def cleared_green_positions(image):
    rows, columns = np.where(image[:, :, 1] == 251)
    return set(zip(rows.tolist(), columns.tolist()))


bit_positions = []
for byte_index in range(MAX_BYTES):
    baseline = cleared_green_positions(oracle("\x00" * (byte_index + 1)))

    for mask in (0x40, 0x20, 0x10, 0x08, 0x04, 0x02, 0x01):
        probe = "\x00" * byte_index + chr(mask)
        probe_positions = cleared_green_positions(oracle(probe))
        difference = baseline - probe_positions
        if len(difference) != 1:
            raise RuntimeError(
                f"bit position is not unique: byte={byte_index}, mask={mask:#x}"
            )
        bit_positions.append(difference.pop())


original = np.array(Image.open(ORIGINAL_IMAGE).convert("RGB"))
stego = np.array(Image.open(STEGO_IMAGE).convert("RGB"))

decoded = bytearray()
position_index = 0

for _ in range(MAX_BYTES):
    bits = ["0"]  # ASCII 最高位

    for _ in range(7):
        row, column = bit_positions[position_index]
        position_index += 1

        cover_green = int(original[row, column, 1])
        stego_green = int(stego[row, column, 1])
        encoded_zero = cover_green & ~(1 << 2)
        encoded_one = cover_green | (1 << 2)

        if stego_green == encoded_zero:
            bits.append("0")
        elif stego_green == encoded_one:
            bits.append("1")
        else:
            raise RuntimeError(f"unexpected pixel value at {(row, column)}")

    decoded.append(int("".join(bits), 2))
    if decoded.endswith(b"}"):
        break

print(bytes(decoded))
```

目标图所用封面可通过反向图片搜索定位到 [Pixiv 作品 97558083](https://www.pixiv.net/artworks/97558083)。把原图裁剪、缩放到与目标完全一致后再比较绿色通道；尺寸或压缩方式不一致会让逐像素对照失效。

最终得到：

```text
hgame{4_New_Type_1mg_Steg4n0graphy}
```

官方 PDF 只描述了探测思路；完整按位探测细节和结果由官方 GitHub 仓库收录的参赛者 Week4 题解交叉核验。

## 方法总结

固定随机种子会把“随机位置 LSB”退化成可重复的固定位置序列，而在线编码接口正好提供选择明文 oracle。用全零基线与单比特探测做集合差，可以逐位恢复位置而无需猜测 PRNG。若写 `0/1` 是对封面位做清除/置位，则还必须取得像素对齐的原始封面；仅看隐写图无法区分原位本来就是 `0` 还是 `1`。
