# 公众号原稿

## 题目简述

附件是一份 DOCX 文档。DOCX 本质上是遵循 OOXML 目录结构的 ZIP 包，除正文、样式和媒体文件外，还可以携带自定义 XML。该附件在 `docProps` 中额外放置了不属于常规属性文件的 `gift.xml`。

## 解题过程

不要只在 Word 可见正文中搜索。将文件复制为 `.zip` 后解压，或直接使用压缩软件打开 DOCX，检查包内目录：

```text
docProps/
├── app.xml
├── core.xml
└── gift.xml
```

`app.xml` 与 `core.xml` 是常见的文档属性文件，而 `gift.xml` 是异常新增项。读取 `docProps/gift.xml`，其内容直接是 flag：

```text
0xGame{omg!Y0u_f0und_m3!_C0ngr4tul4t10ns!}
```

## 方法总结

- 核心技巧：把 Office Open XML 文档视为 ZIP 容器，检查可见正文之外的包内文件。
- 识别信号：DOCX、XLSX、PPTX 等 OOXML 文件可被直接解包；标准目录中出现命名突兀的自定义 XML 时应优先检查。
- 复用要点：不要只做文本提取，还要比较包内结构与常见 OOXML 结构，关注 `docProps`、自定义 XML、媒体和关系文件中的异常内容。
