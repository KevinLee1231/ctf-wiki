# DownUnderCTF 2021 - Retro

## 题目简述

附件 `og.jpg` 是一张能够正常显示的旧比赛 Logo，题面提示它还“隐藏”了秘密。像素内容与解法无关，flag 被写入 JPEG 的作者类元数据；因此图片本身没有需要长期保留的视觉证据，关键字段直接转写到正文即可。

## 解题过程

先在不重编码图片的情况下读取元数据：

```bash
exiftool og.jpg
```

输出中的 `Artist`/`Authors` 字段直接包含：

```text
DUCTF{sicc_paint_skillz!}
```

也可以用 Pillow 验证。JPEG EXIF 标签 315 是 `Artist`；该文件的 Windows `XPAuthor` 字段中也保存了同一段 UTF-16LE 文本：

```python
from PIL import Image, ExifTags

image = Image.open("og.jpg")
exif = image.getexif()

for tag_id, value in exif.items():
    name = ExifTags.TAGS.get(tag_id, str(tag_id))
    if name in {"Artist", "XPAuthor"}:
        print(name, value)
```

不需要对像素做 LSB、颜色通道或压缩域分析。

## 方法总结

图片隐写的最低成本检查应从容器和元数据开始。题面提到“原始制作”“作者”或“旧 Logo”时，优先查看 EXIF、XMP、IPTC 和 Windows XP 字段，再决定是否深入像素。元数据属于文件的一部分，重新保存图片可能会清空它，因此分析前应保留原件并记录哈希。
