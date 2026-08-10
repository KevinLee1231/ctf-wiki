# What is a Word

## 题目简述

附件是 what_is_a_word.docx。正文中的诗只是填充内容，flag 藏在 Word 文档包内的一张媒体图片中。DOCX 本质上是遵循 Open Packaging Conventions 的 ZIP 容器，可以不依赖 Word 界面直接检查内部资源。

## 解题过程

先确认文件头为 ZIP 的 50 4B 03 04，然后列出包内文件。图片通常位于 word/media：

~~~powershell
Copy-Item what_is_a_word.docx what_is_a_word.zip
Expand-Archive what_is_a_word.zip extracted
Get-ChildItem extracted/word/media
~~~

包中有 image1.png 与 image2.png。逐张查看后，image1 只是装饰性小点；image2 是纯文字截图，直接写着：

~~~text
Maple{a_w0rd_is_on3_z1p!}
~~~

这张图没有必须依赖像素、空间关系或图形才能表达的信息，因此在归档 WP 中转写为文本，不保留弱命名的 image2.png。题目元数据也使用相同的大小写，最终 flag 为：

~~~text
Maple{a_w0rd_is_on3_z1p!}
~~~

## 方法总结

Office 文档首先应当作容器检查，而不是只看应用层可见正文。DOCX 中常见的证据位置包括 word/media、document.xml、relationships、批注、页眉页脚和嵌入对象。归档时，纯文字截图应转写为可搜索文本；只有无法由文字等价表达的视觉证据才值得保留。
