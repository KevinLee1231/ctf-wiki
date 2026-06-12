# author

## 题目简述

博客页面把作者名拼进 `<meta name="author">`，注册用户名可以影响 meta 标签属性。解法利用 username 注入 CSP/属性绕过 `xss-shield.js`，再在文章内容中执行 XSS 外带 cookie。

## 解题过程

### 关键观察

博客页面把作者名拼进 `<meta name="author">`，注册用户名可以影响 meta 标签属性。

### 求解步骤

主要就是绕过xss-shield.js，没有csp
header.php 中 author name 可以注入向 meta 标签中注入空格，可以添加任意属性。
尝试构造成 CSP 拦截 xss.shiled.js 。
注册用户时在 username 中写入 csp 内容拦截 xss：
单引号不会被 html 实体化 正好用来包裹 csp 内容。script-src-elem 属性不会影响 unsafe-
inline，只需要设置 article.js 的 URL 即可。
随后用该账号登录发布的 article 不会有任何限制。使用 img onerror 外带：
RCTF{h0w_d1d_u_byp455_th3_w4f_4nd_x55}

## 方法总结

- 核心技巧：meta 属性注入配合 CSP/XSS 绕过。
- 识别信号：用户可控字段进入 `<meta>` 属性，过滤/实体化不完整。
- 复用要点：关注 CSP 指令是否能阻断防护脚本，同时保留目标业务脚本执行。
