# Stalking 1

## 题目简述

WelcomeCTF2021 的 Stalking 1 提供 `stalking.jpg`，要求从图片中找出作者使用的用户名。图像表面只显示多处 “Try Find Me”，真正线索位于元数据。

## 解题过程

检查 JPEG 的 EXIF/XMP 元数据：

```bash
exiftool stalking.jpg
```

关键字段为：

```text
Artist     : situpright899
XP Author  : situpright899
```

同一值也出现在 XMP 的 `dc:creator` 字段中，因此不是图像解码噪声。按题目要求套用 flag 格式：

```text
greyhats{situpright899}
```

图片的可见矩形没有提供额外定位信息，题解直接记录元数据字段即可，不需要保留原图副本。

## 方法总结

OSINT 图片题的第一步应同时检查视觉内容和元数据。EXIF 的 `Artist`、Windows 的 `XPAuthor` 与 XMP 作者字段相互印证时，可以把用户名视为可靠线索；清洗或转码图片可能删除这些字段，所以必须分析原始附件。
