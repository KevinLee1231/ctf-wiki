# scrapbooking

## 题目简述

`warped.png` 表面上是损坏的 PNG，不能直接打开。十六进制检查可见 PNG 魔数多次出现，且相邻起点固定相差 1024 字节；源码表明出题时把三张 PNG 的每个 1024 字节块按图片编号轮流交织，并在短文件末尾补零。这是有意将多个图像载荷碎片化再重组的题目，归入 stego。

## 解题过程

PNG 文件签名为 `89 50 4E 47 0D 0A 1A 0A`。在 `warped.png` 中看到三次签名、间距均为 1024 后，可以确定循环顺序为 image 0、image 1、image 2。恢复时按轮次把每个块追加回对应输出文件：

```python
from pathlib import Path

data = Path("warped.png").read_bytes()
parts = [bytearray(), bytearray(), bytearray()]

for offset in range(0, len(data), 1024):
    index = (offset // 1024) % 3
    parts[index] += data[offset:offset + 1024]

for index, content in enumerate(parts):
    Path(f"image{index}.png").write_bytes(content)
```

恢复出的三张图仅包含可转写的文字片段，依次为：

```text
DUCTF{kn1
tting_wi
th_pngs}
```

按顺序连接得到：

```text
DUCTF{kn1tting_with_pngs}
```

图片本身只承载上述纯文本，因此正文已完整转写，不复制图片或创建资源目录。

## 方法总结

- 核心技巧：用重复文件魔数和等距偏移识别固定大小的轮转交织，再按轮次重建每个原始文件。
- 识别信号：文件无法解析但容器签名在固定间隔多次出现，且题面暗示“剪切、拼贴”时，应检查分块交织而非只尝试 PNG 修复。
- 复用要点：记录块大小、文件数与循环顺序；最后一轮的零填充通常可被 PNG 解码器忽略，但应先恢复完整字节流再判断是否需要裁剪。
