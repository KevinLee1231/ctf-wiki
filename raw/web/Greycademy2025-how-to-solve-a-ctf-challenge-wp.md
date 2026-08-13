# Greycademy2025 How to solve a CTF challenge

## 题目简述

网站只有一个“建设中”页面，没有表单、脚本或后端交互。题目用于提醒参赛者：浏览器渲染结果不等于完整响应，HTML 注释等客户端资源也可能包含敏感信息。

## 解题过程

在浏览器中使用“查看网页源代码”，或直接请求页面：

```bash
curl -s http://target/
```

页面正文之外有两条 HTML 注释：

```html
<!-- TODO: Remember to set up admin panel -->
<!-- caleb:grey{P@ssw0rd123} -->
```

浏览器不会把注释渲染成可见文字，但注释仍随 HTTP 响应完整发送给客户端，任何访问者都可以读取。因此 flag 为：

```text
grey{P@ssw0rd123}
```

## 方法总结

前端隐藏不构成访问控制。排查简单 Web 题时，应检查原始 HTML、开发者工具中的网络响应、静态 JavaScript 和公开资源，而不只观察页面画面。题目机制完全由文本源码表达，没有必要保留浏览器截图，也不应在 WP 中留下已经失效的临时比赛地址。
