# Greycademy2025 babycake

## 题目简述

题目从一张活动海报开始，将 flag 分散在 EXIF 元数据、另一张 PNG 的像素低位和 PNG 结束块后的附加图片中。目标是按线索取得三个片段并按顺序拼接。

## 解题过程

第一张 `part1.jpg` 与原始海报视觉上没有区别，但 `exiftool` 显示两个异常字段：

```text
Make    : 67726579686174737b746834745f
Comment : Check out some of my other works! [short link]
```

将 `Make` 的十六进制解码，得到第一段：

```text
greyhats{th4t_
```

注释中的短链接跳转到一张名为 `cute_cat.png` 的图片。仓库中的 `final_stage_2.png` 与该下载文件一致。先按题目使用的两位像素低位通道提取 10 个字节：

```bash
stegolsb steglsb -r \
  -i final_stage_2.png \
  -o part2.txt \
  -n 2
cat part2.txt
```

输出第二段：

```text
w4s_much_
```

再检查同一 PNG 的容器结构。正常 IEND 位于偏移 `0xf6a22` 前，后面却附加了另一份完整 PNG 签名：

```bash
binwalk final_stage_2.png
dd if=final_stage_2.png of=part3.png bs=1 skip=$((0xf6a22))
```

附加图片上只有手写的第三段：

```text
b1g_c4k3}
```

拼接三个结果得到：

```text
greyhats{th4t_w4s_much_b1g_c4k3}
```

## 方法总结

同一个媒体文件可以同时承载多种隐藏通道。本题先由 JPEG EXIF 给出首段和下一载体，再分别检查 PNG 像素低位与 IEND 后的尾随数据。每层都应记录通道、位数和偏移，不能只写“用在线工具扫出来”。海报、猫图和最终手写片段都没有无法用文本表达的额外视觉机制，因此正文转写结果即可，不保留冗余载体截图；外部短链的关键内容也已完整概括。
