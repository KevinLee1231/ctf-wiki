# GOFTP

## 题目简述

题目提供一个 Go 编写的 FTP Web 客户端。普通用户注册、登录后可以让服务端连接任意 FTP 地址并上传文件；管理员页面 `/admin` 会读取 `/flag`，但中间件要求 Session 中的用户名与 `admin` 不区分大小写相等。公开注册接口同样用 `strings.EqualFold` 禁止注册 `admin`。

真正的用户数据由另一个仅在内网 `9000` 端口开放的 REST 服务管理。前端服务会把正常注册转换成 `PUT /api/user`，但内网接口自身没有禁止管理员用户名。因此目标是借 FTP 被动模式把一段原始 HTTP 请求送到 `127.0.0.1:9000`，注册管理员账号。

## 解题过程

先确认权限检查。管理员中间件只读取 Session：

```go
username, ok := sess.Get("username").(string)
if !ok || !strings.EqualFold(username, "admin") {
    c.Abort()
    return
}
```

外部注册路由会阻止 `admin`，但后端 REST 服务的 `CreateUser` 只检查 JSON 中的 `email`、`username` 和 `password` 是否非空，随后直接写入数据库。因此只要能请求内网接口，就可以建立合法管理员账户。

FTP 上传流程使用 `github.com/dutchcoders/goftp`。该版本解析 PASV 响应后采用其中的端口，却忽略服务器返回的 IP，数据连接仍使用控制连接的主机名。这避免了由 PASV 响应直接指定 `127.0.0.1`，但控制连接和数据连接之间会再次解析主机名，因此可以使用 DNS rebinding：

1. 第一次解析为攻击者服务器，服务端成功建立 FTP 控制连接。
2. 假 FTP 服务依次响应 `USER`、`PASS`、`TYPE`，拒绝 `EPSV`，在 `PASV` 中返回端口 `9000`。
3. 第二次解析相同主机名时返回 `127.0.0.1`。
4. Web 服务把数据连接建立到 `127.0.0.1:9000`，然后把上传文件的原始字节写入该连接。

假 FTP 服务只需实现本次交互涉及的最小命令集，关键响应是：

```text
220 ready
331 Username ok, send password
230 Login successful
200 Type set to Binary
500 EPSV not understood
227 Entering Passive Mode (127,0,0,1,35,40)
150 File status okay
```

其中 $35 \times 256 + 40 = 9000$。上传文件本身是一条完整 HTTP 请求，正文用于创建管理员：

```http
PUT /api/user HTTP/1.1
Host: 127.0.0.1:9000
Content-Type: application/json
Content-Length: 68
Connection: close

{"email":"admin@example.com","password":"123456","username":"admin"}
```

实际使用时必须按正文的真实字节长度填写 `Content-Length`；上面的示意字段需要由客户端重新计算，不能盲目照抄。将 Web 上传表单中的 FTP 地址设为 DNS 重绑定域名，控制端口指向假 FTP 服务。服务端收到 `STOR` 后会把上述内容发往内网 REST API，从而绕过外层注册校验。

然后用 `admin@example.com` 和 `123456` 正常登录。认证结果中的 `Username` 被写入 Session，管理员中间件放行，访问 `/admin` 即可读取：

```text
SCTF{U_Know_th3_FTP_AnD_S0_do_1}
```

## 方法总结

本题的核心是“FTP 数据通道可被改造成跨协议 SSRF”。PASV 返回的 IP 被客户端忽略并没有消除风险，因为数据连接仍会对攻击者控制的主机名再次解析；DNS rebinding 让两次解析分别落到外部 FTP 服务和内网 REST 服务。

防护不能只依赖注册页面的用户名过滤。内网 API 也必须执行同样的权限约束；FTP 目标解析后应固定并校验 IP，拒绝环回、私网和链路本地地址，同时确保控制连接和数据连接使用同一经验证的地址，而不是再次解析可变主机名。
