# A515

## 题目简述

附件 `BLUE.png` 可以正常打开，常规元数据和文件尾检查也没有直接给出隐藏内容。异常之处在于文件包含数量过多的 `IDAT` 块，需要从这些块中筛选并重排真正的图像数据。

## 解题过程

PNG 文件由 8 字节签名和若干块组成，每个块的格式为：

```text
length(4 bytes) || type(4 bytes) || data(length bytes) || CRC(4 bytes)
```

一张 PNG 可以拥有多个 `IDAT` 块，解码器会按文件顺序连接各块的数据。检查 `BLUE.png` 后可以发现 19 个 `IDAT` 块，其中混入了无关块，而且有效块的次序也被打乱。由于每个块连同 CRC 都是完整的，可以直接复制整块，不必重新压缩或计算 CRC。

逐步尝试“已确认序列 + 一个候选块”并观察新增图像区域，可以确定有效顺序：

```text
11, 12, 10, 14, 16, 15, 17, 18, 8, 13, 9
```

下面的脚本按 PNG 结构提取块并重建文件：

```python
from pathlib import Path

data = Path("BLUE.png").read_bytes()
prefix = bytearray(data[:8])
idat_chunks = []
iend = b""
offset = 8
seen_idat = False

while offset < len(data):
    length = int.from_bytes(data[offset:offset + 4], "big")
    chunk_type = data[offset + 4:offset + 8]
    chunk_end = offset + 12 + length
    raw_chunk = data[offset:chunk_end]

    if chunk_type == b"IDAT":
        seen_idat = True
        idat_chunks.append(raw_chunk)
    elif chunk_type == b"IEND":
        iend = raw_chunk
        break
    elif not seen_idat:
        prefix.extend(raw_chunk)

    offset = chunk_end

order = [11, 12, 10, 14, 16, 15, 17, 18, 8, 13, 9]
rebuilt = (
    bytes(prefix)
    + b"".join(idat_chunks[index] for index in order)
    + iend
)
Path("reconstructed-sign.png").write_bytes(rebuilt)
```

重建后的画面与原图完全不同，角色手中的牌子直接给出了 flag：

![按正确顺序重组有效 IDAT 块后的蓝色角色图片，手中标牌写有 N0PS flag](./N0PSctf2025-a515-wp/reconstructed-sign.png)

```text
N0PS{1M4G3_r3C0nsTRuc7i0n}
```

## 方法总结

本题利用的是 PNG 多 `IDAT` 块的顺序语义，而不是简单的文件尾追加或 LSB 隐写。遇到“能正常显示但块数量异常”的 PNG，应解析真实块边界，保留每个块的长度和 CRC，再观察不同排列带来的像素变化。按结构处理比直接搜索字节串 `IDAT` 更稳健，因为压缩数据内部也可能偶然出现相同字样。
