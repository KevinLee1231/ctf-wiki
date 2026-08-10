# 奇怪的图片plus

## 题目简述

题目分为两部分。第一部分要求上传两张 RGB 图片：服务端使用同一个随机密钥分别进行 AES-ECB 加密，将两份密文逐字节异或，再缩放为 $16\times 9$，并检查黑色像素的位置是否与目标图一致。通过检查后会返回第二部分所需的 16 字节密钥 `gift`。

第二部分给出一张由 AES-OFB 加密得到的 `encrypted_flag.png`。明文原图大小为 $200\times150$，背景为黑色，flag 字符绘制在图像中部；已知前 16 字节明文全为零，因此可以从第一块密文恢复 OFB 的内部状态并解出后续像素。

## 解题过程

### 构造 ECB 异或图案

ECB 对相同的 16 字节明文块给出相同密文块。若同一位置的两张图使用完全相同的明文块，则对应密文块异或为零，显示为黑色；若明文块不同，密文异或通常为非零。

RGB 图像每个像素占 3 字节。把 $16\times9$ 的逻辑图案按最近邻放大 48 倍后，每个逻辑像素在水平方向连续占据

$$
48\times3=144=9\times16
$$

字节，而且每行长度 $16\times48\times3=2304$ 也能被 16 整除。这样 AES 分组不会跨越相邻逻辑像素的水平边界，服务端缩回 $16\times9$ 时即可稳定保留设计好的黑色位置。

先令 `image_1` 全黑、`image_2` 全白；在目标黑点处把两张图都设为白色。目标位置会产生相同密文块并异或为黑色，其余位置则由黑、白两种不同明文产生非零结果。

```python
from PIL import Image, ImageDraw

positions = [
    (1, 1), (1, 2), (1, 3), (1, 4), (1, 5), (1, 6), (1, 7),
    (2, 1), (2, 4), (3, 1), (3, 4),
    (4, 1), (4, 2), (4, 3), (4, 4), (4, 5), (4, 6), (4, 7),
    (6, 1), (6, 2), (6, 3), (6, 4), (6, 5), (6, 6), (6, 7),
    (7, 1), (7, 4), (7, 7),
    (8, 1), (8, 4), (8, 7),
    (9, 1), (9, 4), (9, 7),
    (11, 2), (11, 3), (11, 7),
    (12, 1), (12, 4), (12, 7),
    (13, 1), (13, 4), (13, 7),
    (14, 1), (14, 5), (14, 6),
]

image_1 = Image.new("RGB", (16, 9), "black")
image_2 = Image.new("RGB", (16, 9), "white")
draw_1 = ImageDraw.Draw(image_1)
draw_2 = ImageDraw.Draw(image_2)

for position in positions:
    draw_1.point(position, fill="white")
    draw_2.point(position, fill="white")

size = (48 * 16, 48 * 9)
image_1.resize(size, Image.Resampling.NEAREST).save("image_1.png")
image_2.resize(size, Image.Resampling.NEAREST).save("image_2.png")
```

将这两张图片提交后，服务端返回：

```text
8693346e81fa05d8817fd2550455cdf6
```

它应按十六进制解码，而不是把 32 个可见字符直接当作 AES 密钥：

```python
key = bytes.fromhex("8693346e81fa05d8817fd2550455cdf6")
```

### 利用已知明文恢复 OFB 状态

OFB 第一轮输出和第一块密文满足：

$$
O_1=E_K(IV),\qquad C_1=P_1\oplus O_1.
$$

原图左上角为纯黑色，所以 $P_1=0^{128}$，从而 $C_1=O_1$。下一轮密钥流是 $O_2=E_K(O_1)=E_K(C_1)$。因此把 `C_1` 当作一个新 OFB 解密器的 IV，便能从 `C_2` 开始恢复所有剩余明文；最后在前面补回 16 个零字节。

```python
import struct
from Crypto.Cipher import AES
from PIL import Image


def image_to_bytes(image):
    data = bytearray()
    for y in range(image.height):
        for x in range(image.width):
            data.extend(struct.pack("BBB", *image.getpixel((x, y))))
    return bytes(data)


def bytes_to_image(data, width, height):
    image = Image.new("RGB", (width, height))
    for y in range(height):
        for x in range(width):
            offset = (y * width + x) * 3
            image.putpixel((x, y), struct.unpack("BBB", data[offset:offset + 3]))
    return image


key = bytes.fromhex("8693346e81fa05d8817fd2550455cdf6")
ciphertext = image_to_bytes(Image.open("encrypted_flag.png").convert("RGB"))

known_first_block = b"\x00" * 16
state_after_first_round = bytes(
    c ^ p for c, p in zip(ciphertext[:16], known_first_block)
)
cipher = AES.new(key, AES.MODE_OFB, iv=state_after_first_round)
plaintext = known_first_block + cipher.decrypt(ciphertext[16:])
bytes_to_image(plaintext, 200, 150).save("decrypted_flag.png")
```

运行后即可在 `decrypted_flag.png` 中直接读取 flag。官方 PDF 与可检索到的[参赛者题解](https://pythok.icu/2024/02/25/2024-Hgame-week2-wp-crypto/)均给出了完整恢复方法和 `gift`，但没有转写最终图片中的具体字符串；在缺少原始 `encrypted_flag.png` 的情况下无法可靠复原该字符串，因此这里不臆造 flag。

## 方法总结

- 第一部分利用 ECB 的确定性：相同明文块必然产生相同密文块，异或后为全零。
- RGB 像素是 3 字节，构图时必须同时照顾像素边界、16 字节 AES 分组边界与服务端缩放行为；48 倍最近邻放大正好提供稳定对齐。
- OFB 不直接加密明文，而是生成可异或的密钥流。只要已知一个完整明文块，就能恢复对应密钥流状态，并从下一块继续解密。
- 题解应区分“已经由材料确认的解法”和“材料未给出的最终输出”；缺少附件时保留可复现步骤比猜测 flag 更可靠。
