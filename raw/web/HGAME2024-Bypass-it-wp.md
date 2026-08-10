# Bypass it

## 题目简述

登录页提供注册入口，但点击后浏览器弹出“当前不允许注册”并跳回登录页。限制只由注册页面末尾的一段 JavaScript 实现，真正的 `register.php` 后端接口仍然接受注册请求，因此这是客户端限制绕过而不是服务端鉴权突破。

## 解题过程

直接请求 `/register_page.php` 并查看 HTML，可以看到表单仍然完整存在：

```html
<form action="register.php" method="post">
  <input type="text" name="username">
  <input type="password" name="password">
  <input type="submit" name="register" value="注册">
</form>

<script>
alert("很抱歉，当前不允许注册");
top.location.href = "login.html";
</script>
```

脚本只影响浏览器端展示，没有阻止表单页面返回，也没有改变后端接口。可以采用两种方式绕过。

第一种是在站点权限中临时禁止 JavaScript，重新打开注册页并提交表单。提交后再恢复 JavaScript，用新账户正常登录。

更直接的方法是跳过页面，向后端发送 POST 请求：

```bash
curl -i 'http://challenge.host/register.php' \
  --data 'username=player&password=strong-password'
```

表单中的 `register` 按钮参数可以省略，因为后端并未解析它；真正需要的是 `username` 和 `password`。响应正文出现“注册成功”后，回到登录页使用该账户登录即可。

PDF 中的浏览器设置、HTML 和终端截图都只承载可转写的界面文字与请求内容，因此正文已完整表达其信息，没有保留截图。

## 方法总结

- 核心技巧：区分客户端 JavaScript 限制与服务端访问控制，直接调用仍然开放的注册接口。
- 识别信号：页面通过 `alert`、跳转或禁用按钮声称功能不可用，但 HTML 中仍暴露表单 action、字段名和请求方法。
- 复用要点：浏览器端校验不能替代服务端授权；遇到前端阻断时应检查原始 HTML 和网络请求，而不是只按界面流程操作。
