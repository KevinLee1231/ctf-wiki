---
type: technique
tags: [web, ssrf, url-parser, wrapper, normalization]
skills: [ctf-web]
raw:
  - ../raw/web/polyglot-url-tricks-and-ssrf-leaks.md
  - ../raw/web/parser-wrapper-and-legacy-ssrf-tricks.md
updated: 2026-07-27
---

# URL Parser, Wrapper and SSRF Filter Differential

## 适用场景

过滤器、URL 库、DNS 解析器和实际请求客户端对 scheme、authority、userinfo、IP 表示或多次解码理解不同，可让被允许的输入最终访问内网、元数据或非 HTTP 资源。

## 识别信号

- 应用先验证 URL，再由另一库/服务实际发起请求。
- 重定向、userinfo、IPv6、混合进制 IP、DNS rebinding 或 wrapper 改变终点。
- `gopher/file/dict/php` 等 scheme 在某一层被接受。

## 最小证据

- 分别记录校验层看到的 host/scheme 与最终连接地址。
- 使用自控服务器记录 DNS、HTTP 和重定向链。
- 内网访问应有可重复响应、OOB 回连或状态差分。

## 解法骨架

1. 构建 parser matrix：原始字符串、规范化 URL、解析 host、DNS 结果和最终 socket。
2. 单独测试编码、userinfo、IP 形式、重定向与 scheme。
3. 找到视图差异后最小化 payload，并固定 DNS TTL/重绑定窗口。
4. 访问目标资源，再按响应协议提取 token、凭据或内部 API 能力。

## 关键变体

- Parser differential：验证库和请求库拆分 authority 不同。
- Redirect/rebinding：校验时与连接时终点不同。
- Wrapper/protocol smuggling：URL 进一步生成非 HTTP 字节流。

## 常见陷阱

- 只看 URL 字符串，没有记录最终连接地址。
- 公网回显代理被误判为内网 SSRF。
- DNS 缓存和重试导致 rebinding 不稳定。

## 关联技巧

- [polyglot-url-tricks-and-ssrf-leaks.md](polyglot-url-tricks-and-ssrf-leaks.md)
- [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md)
- [request-view-normalization-differentials.md](request-view-normalization-differentials.md)

## 原始资料

- [polyglot-url-tricks-and-ssrf-leaks.md](../raw/web/polyglot-url-tricks-and-ssrf-leaks.md)
- [parser-wrapper-and-legacy-ssrf-tricks.md](../raw/web/parser-wrapper-and-legacy-ssrf-tricks.md)
