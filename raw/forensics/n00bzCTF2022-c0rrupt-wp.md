# c0rrupt

## 题目简述

附件本应是一张 PNG，但文件签名和 `IHDR` 中的尺寸字段均被破坏。需要依据 PNG 固定结构与 CRC 校验恢复文件，而不是靠猜测画布大小。

## 解题过程

先把前 8 字节改为标准 PNG 签名：

```text
89 50 4e 47 0d 0a 1a 0a
```

`IHDR` 数据固定为 13 字节，其中前 8 字节是宽、高。保留位深、颜色类型等其余 5 字节，枚举合理尺寸并对 `IHDR` 类型字节与数据计算 CRC32；与文件中的 `0xec891d43` 相等时，唯一匹配为宽 1357、高 641。写回两个大端 32 位整数后，图片可以正常解码。

```python
import struct, zlib

tail = ihdr_data[8:]
for width in range(1, 3000):
    for height in range(1, 2000):
        data = struct.pack(">II", width, height) + tail
        if zlib.crc32(b"IHDR" + data) & 0xffffffff == 0xec891d43:
            print(width, height)
```

修复后的图像显示：

```text
n00bz{h0w_7he_h41l_d1d_y0u_f1x_th1s?!!}
```

仓库 README 把其中的 `7he` 写成了 `7h3`；这里以修复后的官方附件实际画面为准。

## 方法总结

PNG 修复应优先利用签名、块长度和 CRC 这类结构约束。CRC 不只是完整性字段，也能在未知字段范围有限时充当确定性 oracle，从而避免凭视觉比例盲猜尺寸。
