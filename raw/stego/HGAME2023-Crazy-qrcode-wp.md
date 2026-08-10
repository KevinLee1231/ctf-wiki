# Crazy_qrcode

## 题目简述

附件首先给出一张无法直接扫描的二维码。修复后可取得压缩包密码；压缩包中又包含 25 张被旋转的二维码碎片和一个记录旋转次数的数组。需要先恢复二维码格式信息，再按数组还原并拼接碎片。

## 解题过程

第一张二维码的主体数据没有被破坏，异常来自格式信息。二维码格式信息由纠错级别和掩码模式组成，共有 $4\times8=32$ 种组合。把图片导入 [QrazyBox](https://merri.cx/qrazybox/)，在工具列表中选择 `Brute-force Format Info Pattern`，枚举格式信息后切回 `Decode Mode` 解码，可得到：

```text
QDjkXkpM0BHNXujs
```

这就是压缩包密码。QrazyBox 在这里完成的并非普通扫码，而是枚举损坏的格式信息；即使不使用该网页，也可以对四种纠错等级和八种掩码逐一重写格式位后尝试解码。

解压后得到 25 张等尺寸图片和以下数组：

```text
[1, 2, ?, 3, ?, 0, 3, ?, ?, 3, ?, 0, 3, 1, 2, 1, 1, 0, 3, 3, ?, ?, 2, 3, 2]
```

碎片中能看出三个定位块的局部轮廓。把数组理解为每张图片需要旋转的四分之一圈数后，可以据定位块方向判断所有 `?` 都应取 `2`。按行优先顺序把 25 张图片拼成 $5\times5$ 网格：

```python
from pathlib import Path
from PIL import Image

rotations = [
    1, 2, 2, 3, 2,
    0, 3, 2, 2, 3,
    2, 0, 3, 1, 2,
    1, 1, 0, 3, 3,
    2, 2, 2, 3, 2,
]

tiles = [Image.open(Path("tiles") / f"{index}.png") for index in range(1, 26)]
width, height = tiles[0].size
result = Image.new("RGB", (width * 5, height * 5), "white")

for index, (tile, turns) in enumerate(zip(tiles, rotations)):
    # 若附件编号采用相反旋转约定，将 90 改为 -90。
    restored = tile.rotate(90 * turns, expand=False)
    result.paste(restored, ((index % 5) * width, (index // 5) * height))

result.save("restored-qrcode.png")
```

扫描拼好的二维码，得到：

```text
hgame{Cr42y_qrc0de}
```

二维码有纠错能力，少量非关键模块方向不完全正确时仍可能成功解码，但定位块、时序线和格式信息必须基本一致。最终结果由 [HGAME2023 官方题解仓库](https://github.com/vidar-team/HGAME2023_Writeup) 收录的参赛者 Week2 题解交叉核验。

## 方法总结

处理损坏二维码时应分层判断：先检查定位块和整体网格，再检查格式信息，最后才考虑数据码字损坏。格式信息只有 32 种组合，适合穷举；碎片拼接则可利用定位块方向和给定旋转数组恢复。纠错能力可以容忍少量残缺，却不能替代正确的几何排列。
