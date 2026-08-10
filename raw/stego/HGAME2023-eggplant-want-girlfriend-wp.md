# eggplant_want_girlfriend

## 题目简述

附件是一张 PNG，但 IHDR 的 CRC 校验失败。题目通过篡改图像高度把底部内容藏到正常显示范围之外；根据 IHDR 中保存的 CRC 枚举正确高度并修复字段后，图像底部会出现 flag。

## 解题过程

PNG 的 IHDR 数据包含宽度、高度、位深、颜色类型、压缩方式、过滤方式和隔行方式，紧随其后的 CRC 是对 `b"IHDR" + IHDR数据` 计算得到的。附件宽度为 512，显示高度被改坏；无需盲目修改任意值，可以枚举高度并匹配原 CRC：

```python
from pathlib import Path
import zlib

source = Path("eggplant.png")
data = bytearray(source.read_bytes())

if data[:8] != b"\x89PNG\r\n\x1a\n" or data[12:16] != b"IHDR":
    raise ValueError("not a PNG with IHDR as the first chunk")

width_bytes = bytes(data[16:20])
fixed_tail = bytes(data[24:29])
stored_crc = int.from_bytes(data[29:33], "big")

for height in range(1, 5000):
    height_bytes = height.to_bytes(4, "big")
    ihdr = width_bytes + height_bytes + fixed_tail
    candidate_crc = zlib.crc32(b"IHDR" + ihdr) & 0xFFFFFFFF

    if candidate_crc == stored_crc:
        print(f"height = {height}")
        data[20:24] = height_bytes
        Path("eggplant-fixed.png").write_bytes(data)
        break
else:
    raise RuntimeError("height not found")
```

枚举得到正确尺寸：

```text
512 x 706
```

打开修复后的图片，底部文字为：

```text
hgame{e99p1ant_want_a_girlfriend_qq_524306184}
```

Windows 图片查看器可能会容忍错误高度并显示更多内容，但依赖容错行为并不稳定；依据 CRC 恢复原始 IHDR 才能得到可验证、跨平台的结果。

## 方法总结

- 核心技巧：利用 PNG IHDR CRC 作为约束，枚举被篡改的宽高字段。
- 关键偏移：宽度在字节 `16:20`，高度在 `20:24`，IHDR CRC 在 `29:33`，均使用大端序。
- 复用要点：PNG CRC 报错且图像仍能部分显示时，应优先检查 IHDR 尺寸；修复后还要确认计算 CRC 与文件保存值一致。
