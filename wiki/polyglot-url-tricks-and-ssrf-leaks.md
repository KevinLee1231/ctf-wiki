---
type: family
tags: [web, family, url-parser, ssrf, polyglot, crlf]
skills: [ctf-web]
raw:
  - ../raw/web/polyglot-url-tricks-and-ssrf-leaks.md
updated: 2026-06-12
---

# Polyglot, URL Tricks and SSRF Leaks

## 作用边界

本页是 URL parser 差异、polyglot 上传、SSRF credential leak 和特殊协议 payload 的二级 family。它比 [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) 更窄：只处理输入本身同时满足多种解析器、scheme 或协议语义的情况。

共同判断点是：同一个字符串或文件，在过滤器、业务层、网络客户端和下游解析器中被解释成不同资源。

## 识别信号

- URL 过滤使用 `startswith`、黑名单、正则、host/scheme 手写解析或 path normalize。
- 后端网络客户端支持跳转、userinfo、IPv6、空 host、gopher/file、CRLF header 注入或二次解析。
- 上传文件需要同时满足扩展名/MIME/magic/解析器语义，例如 WAV polyglot。
- SSRF 回连中会带上凭据、token、metadata header 或内部服务响应。

## 最小证据

- 同一 payload 在过滤器和实际请求库中的解析结果：scheme、host、port、path、userinfo、fragment。
- 一个可观测回连或内部响应，能证明请求实际到达预期地址。
- 对 polyglot 文件，至少通过外层校验和内层解析器各一次。
- 对 CRLF/gopher 等 payload，记录原始字节，避免 URL 编码层破坏协议。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| 多斜杠/空 host URL | 过滤器和请求库是否对 host/path 分歧 | 构造最小 parser differential 样本 |
| userinfo/IPv6/编码绕过 | allowlist 是否只看字符串前缀或后缀 | 转 [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md) |
| gopher/no-host scheme | 客户端是否允许非 HTTP scheme 和原始 payload | 内部 Redis/MySQL/HTTP 协议注入 |
| CRLF header/method smuggling | 用户字段是否进入请求行或 header | 控制 method、Host、User-Agent 或 body |
| SSRF credential leak | 出站请求自动附带 cookie、Authorization、metadata token | 把回连服务记录为证据，再进入认证链 |
| WAV/媒体 polyglot 上传 | 文件同时满足扩展名、magic 和解析器结构 | 转 [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md) |
| XSLT `math:random()` seed | 解析器内部随机数可预测 | 转 [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md) |
| DNS rebinding | host 校验和实际访问之间存在再次解析 | 转 [dns.md](dns.md) 补 TTL/解析证据 |

## 合并与拆分结论

- 保留为 family：raw 覆盖 URL、协议、polyglot 文件和 SSRF 泄露，单一 technique 命名无法准确覆盖。
- 不合并进 `parser-wrapper-and-legacy-ssrf-tricks.md`：该页负责所有二段解析器，本页负责 payload 自身的多解释和 URL/SSRF 专项绕过。
- 不拆成 `gopher-ssrf.md`、`url-parser-bypass.md` 等小页：当前 raw 更适合保留为紧凑路由表。

## 常见误判

- 只在浏览器里看 URL 解析，没复现服务端库的真实解析。
- SSRF 只验证外网回连，没检查是否带凭据或能访问内网。
- gopher/CRLF payload 经过多次编码后，目标收到的字节已经不同。
- Polyglot 只过扩展名，不确认内层解析器实际消费了恶意结构。

## 关联页面

- [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md)
- [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)
- [protocol-relay-and-internal-service-injection.md](protocol-relay-and-internal-service-injection.md)
- [dns.md](dns.md)
- [mt-lcg-and-seed-recovery.md](mt-lcg-and-seed-recovery.md)
- [web-tooling.md](web-tooling.md)

## 原始资料

- [polyglot-url-tricks-and-ssrf-leaks.md](../raw/web/polyglot-url-tricks-and-ssrf-leaks.md)
