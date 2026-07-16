# ezEXIF

## 题目简述

服务端接收不超过 2 MB 的 PNG、JPG、JPEG 或 GIF 文件，并通过 ExifTool 读取元数据。源码中的检查条件为：

```text
ImageWidth      = 666
ImageHeight     = 1
Make            = Hacker
Model           = Kali linux
DateTimeOriginal 以 9999:99:99 66:66:66 开头
Description     = motto:I can be better!
```

除 `DateTimeOriginal` 使用 `startswith` 外，其余字段都要求类型和值严格相等。因此既要修改图片真实尺寸，也要伪造四个文本元数据标签。

## 解题过程

先把图片实际尺寸改为 $666\times 1$。最直接的方法是使用 ImageMagick 强制缩放：

```bash
magick input.png -resize 666x1! forged.png
```

PNG 的宽、高不是普通 EXIF 文本，而是 IHDR 数据块中的两个 4 字节大端整数：宽度位于文件偏移 `0x10` 至 `0x13`，高度位于 `0x14` 至 `0x17`。因此宽度 666 应写为 `00 00 02 9A`，高度 1 应写为 `00 00 00 01`。

若直接用十六进制编辑器修改 IHDR，还必须重新计算该数据块的 CRC；否则图片可能无法解析。使用图像工具生成 $666\times 1$ 的文件可以自动完成这一工作。

随后用 ExifTool 写入其余字段：

```bash
exiftool -overwrite_original \
  -Make="Hacker" \
  -Model="Kali linux" \
  -DateTimeOriginal#="9999:99:99 66:66:66" \
  -Description="motto:I can be better!" \
  forged.png
```

正常日期字段会经过格式校验，而题目要求的月份、日期和时间均超出合法范围。`#=` 表示按原始值写入，可绕过 ExifTool 的日期打印转换与常规格式限制。

提交前可用与服务端一致的 JSON 输出检查标签名和值：

```bash
exiftool -json -s3 -q forged.png
```

确认六个字段完全匹配后上传，服务端返回：

```text
0xGame{sometimes_0ur_eYes_may_che@t_us!!!}
```

## 方法总结

本题同时考查图片结构与元数据伪造。`ImageWidth`、`ImageHeight` 来自图片本身，不能只添加同名文本标签；PNG 手改 IHDR 时还要处理 CRC。异常日期则需要用 ExifTool 的原始值写入语法。最后应按服务端实际使用的标签名逐项核对，避免因为 `Model`、`Description` 等字段映射错误而失败。
