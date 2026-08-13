# Greycademy2025 cosmic_bit_flip

## 题目简述

题目给出一张无法正常显示的 `flag.png`，并提示文件遭遇了一次 cosmic bit flip。目标是定位 PNG 结构中被翻转的位，恢复图片后读取 flag。

## 解题过程

`file` 能识别 PNG，但 `pngcheck` 在解压 IDAT 后报告非法行过滤器。IHDR 声明宽高为 `1029 × 242`、位深 8、颜色类型 2，即每个像素应占 3 字节；然而合并并解压所有 IDAT 后得到 996314 字节，恰好满足：

```text
996314 = 242 × (1 + 1029 × 4)
```

每行多出的 1 字节是 PNG 行过滤器，因此实际像素是每点 4 字节的 RGBA。PNG 的颜色类型 6 表示 RGBA，而文件中是 2；两者二进制分别为 `0b110` 与 `0b010`，确实只差一个比特。修复颜色类型后还必须重算 IHDR CRC：

```python
import struct
import zlib

data = bytearray(open("flag.png", "rb").read())
assert data[25] == 2

data[25] = 6
crc = zlib.crc32(b"IHDR" + bytes(data[16:29])) & 0xffffffff
data[29:33] = struct.pack(">I", crc)

open("repaired.png", "wb").write(data)
```

修复后文件通过 `pngcheck`，尺寸仍为 `1029 × 242`。可见 RGB 是随机噪声，但 alpha 通道中清晰写着 flag；使用 ImageMagick 单独导出透明度即可：

```bash
convert repaired.png -alpha extract alpha.png
```

alpha 图中的文字为：

```text
flag{b1t_fl1p_m4d3_my_fl4g_tr4nsp4r3n7}
```

## 方法总结

遇到“能识别但无法解码”的 PNG，不应只改文件签名。应同时核对 IHDR、IDAT 解压长度、每行过滤器字节和 CRC。本题通过行长度反推出每像素 4 字节，再结合“单比特翻转”提示唯一确定颜色类型应从 RGB 改为 RGBA；最后检查各通道，才能从 alpha 中取得真正内容。
