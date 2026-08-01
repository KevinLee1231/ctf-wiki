# Who Am I?

## 题目简述

附件是一个 `.docx` 文档，题目询问作者。DOCX 本质上是 ZIP 容器，作者信息保存在包属性中，不需要从正文猜测身份。

## 解题过程

可直接用 ExifTool：

```text
exiftool "Who Am I.docx"
```

也可以把 DOCX 当 ZIP 解压并查看 `docProps/core.xml`。本地逐项检查得到：

```xml
<dc:creator>Ryan Sketchy</dc:creator>
```

文档正文只有“没有名字、猜不到作者”一类干扰文本；`core.xml` 中的 `creator` 才是题目要求的证据。最终答案为：

```text
byuctf{Ryan Sketchy}
```

## 方法总结

Office Open XML 文档的元数据与可见正文相互独立。处理 DOCX 时应同时检查 `docProps/core.xml`、`docProps/app.xml` 以及媒体、批注等包内对象；本题的决定性字段是 Dublin Core 的 `dc:creator`。
