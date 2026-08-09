# sneaky_png

## 题目简述

PNG 能正常显示，但 `IEND` 结束块后仍附加了文本数据。图像内容只是掩护，载荷位于格式规定的逻辑文件末尾之后。

## 解题过程

`exiftool` 会报告：

```text
Warning: [minor] Trailer data after PNG IEND chunk
```

在十六进制编辑器中定位 `IEND`，跳过该块的类型、数据和 CRC 后读取剩余字节；也可以先用 `strings` 快速确认可打印内容。尾随数据为：

```text
n00bz{sn34ky_str1ngs_c0mm4nd5!}
```

## 方法总结

“能打开”只说明解码器忽略了不影响显示的附加数据。分析媒体文件时，应检查解析器警告、标准结束标志之后的尾随字节以及文件实际长度，而不是只看像素内容和 EXIF 字段。
