# ezHTTP

## 题目简述

网页会根据一系列 HTTP 请求头逐步返回提示，要求客户端伪造来源站点、浏览器标识和本地访问地址。满足全部条件后，响应头中出现一个 JWT；flag 存放在 JWT 的载荷字段中，不需要破解签名密钥。

## 解题过程

使用 Burp Suite、Yakit 或 `curl` 重放请求，并按响应提示依次加入以下请求头：

```http
Referer: vidar.club
User-Agent: Mozilla/5.0 (Vidar; VidarOS x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36 Edg/121.0.0.0
X-Real-IP: 127.0.0.1
```

第三步必须使用 `X-Real-IP`。前一步响应中的 Hint 明确提示不要使用常见的 `X-Forwarded-For`，说明后端只读取特定代理头来判断请求是否来自本机。

条件全部满足后，响应头会增加：

```http
Authorization: Bearer <JWT>
```

JWT 由以点分隔的 header、payload 和 signature 三部分组成。这里只需对 payload 做 Base64URL 解码，而不是“解密 JWT”。官方题解中的载荷解码后包含：

```json
{
  "F14g": "hgame{HTTP_!s_1mP0rT4nt}"
}
```

签名只用于验证令牌没有被篡改，不妨碍读取明文载荷。

## 方法总结

- 核心技巧：根据响应差异逐步伪造 `Referer`、`User-Agent` 和 `X-Real-IP`，再解析响应头中的 JWT。
- 识别信号：服务端连续提示缺少特定 HTTP 头，最后返回 `Authorization: Bearer`。
- 复用要点：代理来源头没有统一可信语义，必须确认应用实际读取的是哪一个；JWT payload 通常只是编码，不应把 Base64URL 解码误称为解密或签名破解。
