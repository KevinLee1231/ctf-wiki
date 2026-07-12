# PasswdStealer

## 题目简述

题目是 SpringBoot 场景下利用 CVE-2024-21733 的 HTTP 请求走私/缓冲区错位问题。附件中的 Docker 只启动 `PasswdStealer-0.0.1-SNAPSHOT.jar`，环境变量 `FLAG` 存放敏感信息；JAR 内嵌 `tomcat-embed-core-9.0.43.jar`，也就是漏洞版本 Tomcat。

外链文章的核心结论是：Tomcat 在读取请求体超时后，如果没有正确执行 `reset()`，`inputBuffer` 的 `limit` 会错位，后续同连接请求可复用残留缓冲区内容。利用需要同时满足超时、保持连接进入下一轮解析、以及有可回显的请求。

参考 URL：https://mp.weixin.qq.com/s?__biz=Mzg2MDY2ODc5MA==&mid=2247484002&idx=1&sn=7936818b93f2d9a656d8ed48843272c0

## 解题过程

### 关键机制

漏洞利用链分成三步：

1. 构造 `Content-Length` 大于真实长度的 `multipart/form-data` 请求，让 Tomcat 在读取上传体时超时并抛出 `IOException`。
2. 避免该异常向上包装成 500。普通 multipart 丢弃请求体时会在 `discardBodyData()` 抛错并导致 `keepAlive=false`；去掉 boundary 结束标记后，异常会被 `readBoundary()` 捕获并包装成 `MalformedStreamException`，最终返回 404。404 不在 `StatusDropsConnection` 中，所以连接继续进入 `Http11Processor.service()` 的下一轮循环。
3. 在缓冲区错位后发送 `TRACE` 请求，让敏感 header 或 body 被拼成 TRACE 请求头并回显。

multipart boundary 规则为：`--boundary` 表示一个 part 开始，`--boundary--` 表示表单结束。题目利用的正是缺失结束标记时的异常路径差异。

### 求解步骤

先发送一个受害者请求，令敏感信息进入同一连接的 `inputBuffer`。随后攻击者发送正常请求推进缓冲区状态，再发送精心裁剪的 multipart 请求触发超时但保持连接。超时返回后，`nextRequest()` 会把错位后的残留内容作为下一条请求解析。

最后构造 TRACE 内容，使残留的 flag header 被覆盖成 TRACE 请求头的一部分。由于 TRACE 会回显收到的请求头，响应体中即可出现 flag。

可用流程如下：

```text
victim request with sensitive header
attacker normal request
attacker malformed multipart request without final boundary
TRACE / HTTP/1.1
Host: target
...
```

如果目标泄露的是 body，也可以先发送大量 CRLF 填充缓冲区，再把 body 覆盖成 TRACE headers。

## 方法总结

- 识别点：SpringBoot + Tomcat 9.0.43 + multipart 上传处理。
- 核心条件：超时后不返回 500、连接保持 keep-alive、后续请求能回显。
- 外链 CVE 分析的关键信息已内化为：`reset()` 被跳过导致缓冲区 `limit` 错位，利用目标是让旧请求内容被下一轮请求解析。
