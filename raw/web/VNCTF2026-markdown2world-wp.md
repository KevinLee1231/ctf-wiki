# Markdown2world

## 题目简述

题目提供 Web 页面把用户提交的 Markdown 转成 Word 文档并下载，后端调用 pandoc 生成 `.docx`，但转换过程没有启用足够的沙箱或资源访问限制。pandoc 处理 Markdown 图片语法时会尝试读取图片目标，并把读取到的资源写入 docx 的媒体目录。

因此攻击目标不是直接命令执行，而是让 pandoc 在服务器侧读取目标 `.so` 或 flag 相关资源。构造包含图片引用的 Markdown，转换后下载生成的 `.docx`，将其作为 zip 解压，在 `word/media/` 一类目录中即可找到被嵌入的 `.so` 文件，文件内容中包含 flag。

## 解题过程

这是 pandoc 资源加载行为和 docx 打包格式的组合。pandoc 在 Markdown 转 Word 时如果不加沙箱，会把图片语法中的目标当作媒体资源加载，所以提交如下 Markdown 内容：

```markdown
![a](<目标 .so 或本地资源路径>)
```

上传后转换成 docx 并下载，将 docx 当作 zip 解压，在媒体文件夹里找到被打包进去的 `.so` 文件，文件内容就是 flag。

## 方法总结

文档转换类 Web 题要关注转换器是否会访问本地文件、远程 URL 或把引用资源复制进输出文件。`docx` 本质是 zip 包，遇到 Markdown/HTML 转 Word 后的异常附件或媒体资源，应直接解压检查 `word/media/`、关系文件和嵌入对象。
