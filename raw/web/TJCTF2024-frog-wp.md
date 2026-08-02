# frog

## 题目简述

题目网站首页只有青蛙内容，但站点根目录公开了 `robots.txt`。其中 `Disallow` 暴露一个所谓秘密目录；该指令只请求搜索引擎不要抓取，并不会对普通 HTTP 客户端实施访问控制。

## 解题过程

依次请求三个路径即可形成完整发现链：

```bash
curl -s 'https://TARGET/robots.txt'
# User-agent: *
# Disallow: /secret-frogger-78570618/

curl -s 'https://TARGET/secret-frogger-78570618/'
```

秘密目录的页面用 CSS 把链接字号设为 0，但 HTML 源码仍包含：

```html
<a href="flag-ed8f2331.txt" style="text-decoration: none;">🐸</a>
```

直接请求该相对路径：

```bash
curl -s 'https://TARGET/secret-frogger-78570618/flag-ed8f2331.txt'
```

得到：

```text
tjctf{fr0gg3r_1_h4rdly_kn0w_h3r_3e1c574f}
```

## 方法总结

- `robots.txt` 是公开索引提示，不是权限系统；把敏感目录写进 `Disallow` 反而会向任何访问者泄漏路径。
- CSS 隐藏只影响视觉呈现，浏览器开发者工具、`curl` 或查看源代码都能看到链接目标。
- 真实保护应在服务器端对目录和文件做认证授权，随机化路径和前端隐藏都只能降低偶然发现概率。
