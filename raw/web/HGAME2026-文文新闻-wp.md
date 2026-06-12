# 《文文。新闻》

## 题目简述

官网题干是“来看看文文今天又带来了什么劲爆新闻，同时来帮助文文发布新闻吧！”。题目没有附件，只有在线靶机。服务是一个前端代理加 Rust 后端业务服务的 Web 题：前端运行 Vite dev server，后端负责 `/api/comment` 等接口处理，flag 会出现在 bot 发出的请求头中。题目关键点有两个：一是 Vite 的任意文件读取漏洞 CVE-2025-30208 可以帮助读取部署路径和源码；二是前端转发请求时不清理危险请求头，而后端只按 `Content-Length` 判断 body 长度，导致前后端 HTTP 解析差异。

从 `/proc/self/environ` 可以知道服务路径包含 `/app/backend` 和 `/app/frontend`。直接用 CVE 读 `main.rs` 不稳定，但可以借助 Vite dev 的 `@fs` 读取后端源码，确认后端存在只信任 `Content-Length` 的解析逻辑。

## 解题过程

先利用 CVE-2025-30208 做任意文件读取，确认前后端目录位置，再通过 `@fs` 读取后端源码。源码里后端读取请求体时只根据 `Content-Length` 截取数据，而前端代理不会过滤 `Transfer-Encoding` 等请求头，因此可以构造 CL/TE 不一致的请求走私。

核心请求形态如下，第一段请求对后端来说没有 body，随后 `bf` 被当作垃圾丢弃，第二个 POST 带一个过大的 `Content-Length`，迫使后端等待同一 TCP 连接上的后续数据：

```http
POST /api/comment HTTP/1.1
Host: localhost
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked
Authorization: token_example

bf
POST /api/comment HTTP/1.1
Host: localhost
Content-Type: application/x-www-form-urlencoded
Content-Length: 300
Authorization: token_example

content=STOLEN_DATA:

0
```

走私成功后，后端会把后续到达同一连接的数据补进第二个请求的 body。如果 bot 的访问恰好复用这条前端到后端的连接，bot 请求头就会被拼接进评论内容，flag 所在的请求头也随之泄露。

这里必须通过前端代理打后端，不能直接打后端端口。请求走私依赖前端和后端之间的同一条 TCP 连接缓冲区，直接访问后端无法截获其他用户或 bot 的请求。

## 方法总结

- 核心技巧：利用 Vite 任意文件读取拿源码，再利用前后端 CL/TE 解析差异做 HTTP request smuggling。
- 识别信号：前端代理不清洗请求头、后端手写 HTTP/body 解析、只信任 `Content-Length` 时，应优先考虑请求走私。
- 复用要点：走私题要确认攻击链路中的连接复用位置；只有前端到后端的连接可被复用时，才能截获 bot 或其他请求。
