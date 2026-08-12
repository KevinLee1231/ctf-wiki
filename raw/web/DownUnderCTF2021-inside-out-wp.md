# DownUnderCTF 2021 - Inside Out

## 题目简述

网站提供代发 HTTP 请求的 `/request` 接口，只禁止目标解析为回环地址，却允许访问容器私网。首页源码还暴露了 `/admin` 路径；利用示例请求返回的 `ip addr` 信息确定服务内部地址，再让代理从内部访问管理端点，即可满足基于 `X-Real-IP` 和本地网段的信任判断。

## 解题过程

### 找到内部管理接口

首页 HTML 中留有注释：

```html
<!-- <a href="/admin">Admin Panel</a> -->
```

直接从公网访问 `/admin` 时，前置代理会写入外部来源的 `X-Real-IP`，不属于应用启动时根据 `eth0` 计算的 `LOCAL_CIDR`，因此返回 403。

### 通过示例响应取得私网地址

首页的“Proxy Example”指向：

```text
/request?url=http://example.com/
```

应用对这个 URL 做了特殊处理，不是真的访问 `example.com`，而是执行 `ip addr` 并把输出放进 JSON 的 `text` 字段。由其中 `eth0` 的 IPv4 地址与掩码可得到当前容器网络地址，例如：

```text
inet 172.x.x.x/16 ... eth0
```

### 用 SSRF 从内部访问 `/admin`

`/request` 的目标检查只调用：

```python
hostname = urlparse(url).hostname
if hostname and is_localhost(hostname):
    return "blocked", 403
```

`is_localhost` 只在 DNS 解析结果是 IPv4 或 IPv6 回环地址时返回真，并未拒绝 RFC 1918 私网地址。于是把示例中取得的内部服务地址代入：

```text
/request?url=http://INTERNAL_IP/admin
```

在赛事部署中应使用该服务内部 HTTP 入口对应的端口。请求从应用所在网络发出，经内部前置代理到达 Flask；`X-Real-IP` 因而是同一 `LOCAL_CIDR` 中的地址。管理端判断如下：

```python
if (
    "X-Real-Ip" not in request.headers
    or is_localhost(request.headers["X-Real-Ip"])
    or ipaddress.ip_address(request.headers["X-Real-Ip"])
       in ipaddress.ip_network(LOCAL_CIDR, False)
):
    print("Local network request!")
else:
    return "forbidden", 403
```

SSRF 响应的 `text` 字段包含管理页面，部署环境中的实际 flag 为：

```text
DUCTF{very_spooky_request}
```

仓库 `challenge.yml` 还接受过复数结尾的兼容值，但 `docker-compose.yml` 注入给实际应用的是上述单数版本。

## 方法总结

本题是典型的“目标地址过滤不完整 + 信任私网来源”SSRF。只禁止 `127.0.0.0/8` 或 `::1` 并不能阻止访问容器网段；服务端授权也不应把可伪造或可由反向代理产生的来源 IP 当作身份。应对解析后的全部地址做私网、链路本地和特殊用途网段校验，并在每次解析与连接之间防止 DNS 重绑定，同时让管理接口使用真正的认证机制。
