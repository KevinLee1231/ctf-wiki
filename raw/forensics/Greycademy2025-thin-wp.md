# Thin

## 题目简述

附件 `thin.img` 是一个约 2 MiB 的 FAT12 文件系统镜像。题目提示文件已被删除；需要先恢复删除项，再识别被伪造的文件头，并从恢复后的 PDF 页面和元数据中取得两段 flag。

## 解题过程

使用 Sleuth Kit 枚举镜像可见删除项 `flag.png`，其元数据地址为 5：

```bash
fls -r -p thin.img
# r/r * 5: flag.png

icat -r thin.img 5 > deleted.bin
xxd -l 16 deleted.bin
```

文件开头为：

```text
25 50 4e 47 2d 31 2e 36 0d 25 e2 e3 cf d3 0d 0a
%PNG-1.6.%......
```

`PNG-1.6` 不是合法 PNG 版本串，却与 PDF 的 `%PDF-1.6` 只差两个字符。仅修复临时副本前四字节，并保留后续内容：

```bash
cp deleted.bin repaired.pdf
printf '%%PDF' | dd of=repaired.pdf bs=1 count=4 conv=notrunc status=none
qpdf --check repaired.pdf
mutool draw -o page-%d.png -r 160 repaired.pdf
exiftool repaired.pdf
```

`qpdf` 确认文件没有语法或流编码错误，且 PDF 只有 1 页。逐页视觉核对后，页面中央文字为：

```text
Part 1: grey{rmb_2_w1p3_
```

`exiftool` 和 PDF Info 字典则显示：

```text
Author : Part 2: y0ur_fr33_sp4c3!!}
```

按顺序拼接两段得到 `grey{rmb_2_w1p3_y0ur_fr33_sp4c3!!}`。页面只含可完整转写的单行文字，因此不保留整页截图。

## 方法总结

删除文件恢复后不能相信扩展名，应同时检查魔数、版本串和内部结构。本题依次考查 FAT12 删除项恢复、四字节签名修复、PDF 页面视觉核对和元数据检查；只看页面会漏掉后半段。最终 flag 为 `grey{rmb_2_w1p3_y0ur_fr33_sp4c3!!}`。
