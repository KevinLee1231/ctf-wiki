# week1where_U_from

## 题目简述

Nginx 配置用三个可控的 HTTP 属性依次鉴权：`X-Forwarded-For` 必须等于 `127.0.0.1`，Cookie `login` 必须为 `1`，请求方法必须为 POST。客户端可直接伪造这些字段，因此所有检查都可绕过。

## 解题过程

根路径提示 `visit /flaggeeee in localhost`。配置实际检查的是 `$http_x_forwarded_for`，并没有验证该头是否由可信反向代理写入，因此请求 `/flaggeeee` 时加入：

```http
X-Forwarded-For: 127.0.0.1
```

响应继续提示访问 `/re3l_flag`。该位置依次检查：

```nginx
if ($http_x_forwarded_for != 127.0.0.1) { ... }
if ($cookie_login != 1) { ... }
if ($request_method != POST) { ... }
```

响应头使用的是自定义 `Cookie: login=0`，而不是浏览器会自动保存的 `Set-Cookie`；因此需要在请求中主动发送 `Cookie: login=1`。最终请求可写为：

```http
POST /re3l_flag HTTP/1.1
Host: target
X-Forwarded-For: 127.0.0.1
Cookie: login=1
Content-Length: 0
```

三个条件同时满足后，Nginx 返回：

```text
0xGame{http_head3r_m2tters}
```

## 方法总结

- 核心方法：伪造 XFF、Cookie 和请求方法，逐项满足 Nginx 中基于客户端可控字段的条件。
- 识别特征：响应连续提示本地来源、登录状态和 POST 方法，且后端直接读取对应 HTTP 头。
- 注意事项：只有经过受信任代理清洗和重写的 XFF 才能用于来源鉴权；`Cookie` 响应头不会像 `Set-Cookie` 一样建立浏览器会话。
