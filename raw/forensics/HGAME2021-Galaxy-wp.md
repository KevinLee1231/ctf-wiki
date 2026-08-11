# Galaxy

## 题目简述

附件是一份网络流量。通过 HTTP 对象导出可以恢复 `galaxy.png`，但该图在部分查看器中无法正常显示。PNG 的 IHDR 高度字段被篡改，导致保存于文件中的 CRC32 与当前块内容不匹配；枚举高度并匹配 CRC 即可恢复原图。

## 解题过程

在 Wireshark 中打开“文件 → 导出对象 → HTTP”，导出来自 `192.168.43.146`、大小约 4748 KB 的 `galaxy.png`。检查 PNG 结构：

- 字节 `12:16` 是块类型 `IHDR`；
- 字节 `16:20` 是宽度；
- 字节 `20:24` 是高度；
- 字节 `24:29` 是位深、颜色类型、压缩、过滤和隔行参数；
- 字节 `29:33` 是 IHDR 的 CRC32。

CRC 覆盖块类型和块数据，因此可以固定其他字段并枚举高度：

```python
import struct
import zlib

data = bytearray(open("galaxy.png", "rb").read())
expected_crc = int.from_bytes(data[29:33], "big")

for height in range(0x10000):
    ihdr = bytes(data[12:20]) + struct.pack(">I", height) + bytes(data[24:29])
    if zlib.crc32(ihdr) & 0xFFFFFFFF == expected_crc:
        print(f"height = {height:#x}")
        data[20:24] = struct.pack(">I", height)
        open("galaxy-repaired.png", "wb").write(data)
        break
else:
    raise RuntimeError("未在枚举范围内找到匹配高度")
```

匹配到的高度是 `0x1000`。修复高度后重新打开图片，可以读到：

```text
hgame{Wh4t_A_W0nderfu1_Wa11paper}
```

## 方法总结

从流量中恢复文件后仍需继续验证文件结构，不能以“成功导出”为终点。PNG 的 CRC 同时是完整性检查和字段恢复约束：当只怀疑宽度或高度等小范围整数时，固定其余 IHDR 字段进行枚举，比凭显示效果猜尺寸更可靠。
