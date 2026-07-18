# week1robots

## 题目简述

网站把隐藏页面路径写入根目录的 `robots.txt`，而 flag 又位于该页面的 HTML 注释中。整个过程不涉及权限绕过，只需按公开线索继续查看源代码。

## 解题过程

访问：

```text
/robots.txt
```

文件内容为：

```text
4ll.html
```

继续访问 `/4ll.html`。页面正文没有直接显示 flag，但查看原始 HTML 可以在注释中的 ASCII 图末尾找到：

```text
0xGame{Rob0t_le4ks_seCr3t}
```

## 方法总结

`robots.txt` 只是给爬虫的抓取建议，本身完全公开，不能承担访问控制或隐藏敏感路径的职责。发现该文件后，还应继续检查列出的页面、HTML 注释、脚本和静态资源，而不能只看浏览器渲染结果。
