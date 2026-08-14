# Read Read Read

## 题目简述

附件是一份看似普通的 DOCX 故事文档，题目提示可以用颜色隐藏消息。目标是检查页眉页脚与文字样式，找出白色字体隐藏的 flag。

## 解题过程

在 Word 中进入页眉编辑区域并全选文字，把字体颜色改为黑色，即可显出隐藏内容。若不依赖 Office 界面，也可把 DOCX 当作 ZIP 解包，检查页眉 XML：

```bash
unzip "The Tale of the Acidic Juice.docx" -d unpacked
```

`word/header1.xml` 中存在如下运行属性与文本：

```xml
<w:r xmlns:w="http://schemas.openxmlformats.org/wordprocessingml/2006/main">
  <w:rPr>
    <w:color w:val="FFFFFF"/>
  </w:rPr>
  <w:t>grey{iNVi5b1e_1nK_w0rKs_h3r3_t0o}</w:t>
</w:r>
```

`FFFFFF` 是白色，文字在白底页眉中不可见，但内容从未加密或删除。最终 flag 为：

```text
grey{iNVi5b1e_1nK_w0rKs_h3r3_t0o}
```

## 方法总结

Office 文档是包含正文、页眉页脚、关系和样式信息的 ZIP/XML 容器。视觉上“空白”不代表没有文字；对可疑 DOCX，应检查页眉页脚、字体颜色、隐藏属性和底层 XML，而不仅是正文页面。
