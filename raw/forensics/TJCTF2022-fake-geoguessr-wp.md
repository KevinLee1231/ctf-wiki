# fake-geoguessr

## 题目简述

题目给出一张雨天缆车湖景照片，并刻意强调相机型号、Circle of Confusion 等信息。画面地点不是解题目标；flag 被拆在 JPEG 的两个元数据字段中。

![附件中的雨天缆车湖景照片，决定性证据实际位于其 JPEG 元数据](TJCTF2022-fake-geoguessr-wp/rainy-cable-car-lake.jpg)

## 解题过程

直接用 ExifTool 枚举全部 EXIF、IPTC 与注释字段：

```bash
exiftool lake.jpg
```

关键输出为：

```text
Copyright : tjctf{thats_a_
Comment   : lot_of_metadata}
```

按字段内容顺序拼接即可得到：

```text
tjctf{thats_a_lot_of_metadata}
```

照片还保留了 `iPhone 6 Plus`、拍摄时间、镜头参数以及 GPS 坐标 `23°51'12.72" N, 120°56'14.64" E`，但这些只是强化“检查元数据”的提示，并不需要据此做地理搜索。

## 方法总结

- 取证题应先检查容器元数据、注释和缩略图，再决定是否需要图像识别或 OSINT。
- ExifTool 会统一展示 JPEG 内多套元数据命名空间，适合避免只查 EXIF 而漏掉 Comment/IPTC。
- 发现 flag 片段后仍应检查其字段边界与顺序，不能把同文件中的所有字符串无条件拼接。
