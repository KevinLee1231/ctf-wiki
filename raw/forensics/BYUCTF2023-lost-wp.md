# BYUCTF 2023 - Lost

## 题目简述

题目原本通过 Box 提供一组照片。每张照片的 EXIF 中包含 GPS 坐标，需要按文件顺序提取并连线，从轨迹形状中读出 flag。

## 解题过程

对每张 JPEG 读取 `GPSLatitude`、`GPSLongitude` 及南北/东西半球标记，把度分秒换成十进制度：

```python
def decimal(dms, ref):
    value = float(dms[0]) + float(dms[1]) / 60 + float(dms[2]) / 3600
    return -value if ref in ('S', 'W') else value
```

按自然文件名顺序保存坐标，使用 OpenStreetMap、Leaflet 或普通散点图依次连线。轨迹形成可读文本：

```text
@CleverUSB
```

所以答案是：

```text
byuctf{@CleverUSB}
```

当前仓库未包含 Box 中的照片，只保留官方 WP 和提取代码；因此坐标点集及最终轨迹图无法从本地仓库复算，本文没有伪造一张示意图冒充原始证据。

## 方法总结

照片取证先保留顺序、原始 EXIF 和坐标符号，再做可视化。若连线结果混乱，应检查文件排序、DMS 转换以及 `S/W` 负号；缺失原始证据时要明确区分官方结论与本地已验证结果。
