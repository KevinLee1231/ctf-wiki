# BYUCTF 2023 - HUUP

## 题目简述

服务用 UDP 接收原始 HTTP 请求，再转发给本机 Flask TCP 服务。每个 UDP 数据报最多读取 1024 字节，数据按来源地址缓存，并在 10 秒后丢弃。源码本想把累计长度限制为 8000 字节，但误写成 `len(messages[key])`；该对象始终是二元素列表，所以长度上限实际未生效。

## 解题过程

发送最小 GET 请求即可：

```python
request = b"GET / HTTP/1.1\r\nHost: 127.0.0.1\r\n\r\n"
sock.sendto(request, (host, port))
data, _ = sock.recvfrom(7000)
```

中间件发现 `\r\n\r\n` 后建立 TCP 连接，但只调用一次：

```python
response = httpsock.recv(10000)
```

一次 `recv` 不保证拿到完整 HTTP 响应；经常只得到以空行结尾的头部。因此若响应以 `\r\n\r\n` 结束，就重复发送同一请求，直到收到正文。

仓库当前 `server.py` 的 `/` 已直接返回 flag，官方 `solve.py` 也只重试根路径：

```text
byuctf{there's_a_reason_we_do_HTTP_over_TCP_fc163467}
```

README 仍描述旧设计：先取 `/endpoints.txt` 再枚举 200 个路径；源码中该入口提示已被注释。真正导致截断的不是 UDP 天生“丢正文”，而是代理对上游 TCP 响应只做了一次 `recv`。

## 方法总结

TCP 是字节流，单次 `recv(n)` 只表示“最多 n 字节”，不表示一条完整 HTTP 消息。代理必须按 `Content-Length`、chunked 编码或连接关闭持续读取；换成 UDP 外壳也不能省略应用层分帧。
