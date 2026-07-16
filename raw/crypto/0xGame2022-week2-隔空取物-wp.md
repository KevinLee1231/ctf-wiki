# week2隔空取物

## 题目简述

附件 `flag.zip` 使用传统 ZipCrypto 加密，压缩包目录中可见目标文件名为 `flag.png`。PNG 的签名和首个 IHDR 块开头具有固定格式，可以提供足够的已知明文，用 `bkcrack` 恢复 ZipCrypto 的三个内部密钥。解密后的 PNG 还被篡改了 IHDR 宽高，需要利用块 CRC 反推正确尺寸。

## 解题过程

PNG 开头 16 字节固定为：8 字节签名、长度为 13 的 IHDR 数据长度，以及块类型 `IHDR`。

```text
89504e470d0a1a0a 0000000d 49484452
```

把这 16 字节构造成已知明文。它们在 PNG 文件中的偏移为 0，然后对传统 ZipCrypto 执行已知明文攻击：

```bash
echo 89504e470d0a1a0a0000000d49484452 | xxd -r -p > png_header
bkcrack -C flag.zip -c flag.png -p png_header -o 0
```

恢复出的三项并不是 ZIP 密码文本，而是 ZipCrypto 的三个 32 位内部密钥：

```text
ab7d8bcd 6ce75578 4de51c12
```

使用密钥直接解密目标条目：

```bash
bkcrack -C flag.zip -c flag.png \
  -k ab7d8bcd 6ce75578 4de51c12 \
  -d flag.png
```

得到的 `flag.png` 仍无法正常显示。PNG 的 IHDR 数据位于偏移 16–28，其中宽、高分别占 4 字节；偏移 29–32 保存对 `IHDR + 13 字节数据` 计算出的 CRC32。题目保留了原 CRC，却破坏了宽高，因此枚举尺寸并与保存的 CRC 比较即可同时定位并修复：

```python
import binascii
import struct

path = "flag.png"
png = bytearray(open(path, "rb").read())

assert png[:8] == b"\x89PNG\r\n\x1a\n"
assert png[12:16] == b"IHDR"

stored_crc = struct.unpack(">I", png[29:33])[0]
fixed = False

for width in range(1, 4001):
    for height in range(1, 4001):
        ihdr = (
            b"IHDR"
            + struct.pack(">I", width)
            + struct.pack(">I", height)
            + png[24:29]
        )
        if binascii.crc32(ihdr) & 0xFFFFFFFF == stored_crc:
            print("width, height =", width, height)
            png[16:20] = struct.pack(">I", width)
            png[20:24] = struct.pack(">I", height)
            open("flag_fixed.png", "wb").write(png)
            fixed = True
            break
    if fixed:
        break
```

匹配到的尺寸为 `1080 × 608`。修复后的图片显示：

```text
0xGame{D0_Y0u_L1k3_89504E47?}
```

## 方法总结

主线是 ZipCrypto 已知明文攻击：只要能确定加密条目中连续的明文字节，就有机会恢复内部状态，而 PNG 固定文件头正好提供了这些字节。解密成功后还要继续检查文件结构；IHDR 的 CRC 覆盖块类型和块数据，保存的正确 CRC 可以反向约束被篡改的宽高。`89504E47` 是 PNG 签名的前四字节，也与最终 flag 相呼应。
