---
type: technique
tags: [forensics, technique]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/3d-printing.md
updated: 2026-05-21
---

# 3D Printing / CAD File Forensics

## 适用场景

附件是 3D 打印、CAD、G-code、PrusaSlicer binary G-code、打印过程视频或罕见图像格式，flag 可能藏在打印路径、注释、缩略图、模型侧视投影或文件 metadata 中。

## 识别信号

- 文件 magic 为 `GCDE`、`qoif`，或扩展名是 `.g`、`.bgcode`、`.gcode`、`.stl`、`.3mf`。
- 文本中有 `G0/G1`、`LAYER_CHANGE`、`TYPE:Custom`、坐标和挤出量 `E`。
- 打印路径投影后可能形成文字；侧视 XZ/YZ 比俯视 XY 更有信息。
- 缩略图或 metadata 中可能嵌入 PNG/JPEG/QOI。

## 最小证据

- 已识别真实格式和压缩块类型，至少导出可读 G-code 或 thumbnail。
- 能从 G-code 中提取坐标序列，并画出 XY/XZ/YZ 至少一个投影。
- 如果是视频题，至少跟踪出喷头或打印床的一段稳定坐标轨迹。

## 解法骨架

1. `file` / `xxd -l 16` 确认 magic；`GCDE` 按 PrusaSlicer binary G-code 解析 block。
2. 解压 GCode block，搜索 `flag`、`ctf`、`secret`、comments、custom section。
3. 导出 thumbnail，特别检查 QOI/QOIF。
4. 从 `G1` 提取 X/Y/Z/E，按挤出动作过滤，画 XY、XZ、YZ 投影。
5. 视频题用 OpenCV 跟踪喷头/床位置，过滤打印层帧，生成 2D histogram。

## 关键变体

| 变体 | 复用重点 |
|---|---|
| PrusaSlicer `.bgcode` | `GCDE` block 结构，GCode/thumbnail block 可能压缩。 |
| QOI thumbnail | 缩略图不是常见 PNG/JPEG 时，按 `qoif` 解析。 |
| G-code comments | flag 常在注释、object name、custom G-code 或 layer marker。 |
| Side projection | XZ/YZ 投影可显示侧面文字，不能只看 XY。 |

## 常见陷阱

- 把 `.g` 当普通文本，忽略 binary G-code 压缩块。
- 只画 XY 俯视图，错过侧面/高度方向文字。
- 没按 extrusion 过滤路径，travel move 污染图像。
- 忽略 thumbnail 和 metadata，它们常比路径更直接。

## 关联技巧

- [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md)
- [blockchain-and-transaction-forensics.md](blockchain-and-transaction-forensics.md)
- [cross-domain-forensics-technique-map.md](cross-domain-forensics-technique-map.md)
- [disk-memory-vm-and-container-forensics.md](disk-memory-vm-and-container-forensics.md)
- [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)

## 原始资料

- [3d-printing.md](../raw/forensics/3d-printing.md)
