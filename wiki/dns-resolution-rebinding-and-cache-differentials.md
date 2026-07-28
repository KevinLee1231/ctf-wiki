---
type: technique
tags: [web, pentest, dns, rebinding, cache, ssrf, toctou]
skills: [ctf-web, ctf-pentest]
raw:
  - ../raw/pentest/dns.md
  - ../raw/web/parser-wrapper-and-legacy-ssrf-tricks.md
  - ../raw/web/SUCTF2026-uriWP.md
updated: 2026-07-28
---

# DNS Resolution Rebinding and Cache Differentials

## 适用场景

安全检查、代理、应用连接和下游客户端在不同时间或不同解析器上解析同一主机名；攻击者可利用 DNS rebinding、split-horizon、缓存/TTL 差异或 TOCTOU，让校验看到公网地址而真实连接命中内网。

## 识别信号

- 应用先解析并校验 URL，等待后再由 HTTP 客户端重新解析连接。
- 黑名单拒绝私网 IP，却保留攻击者可控域名和明显时间窗。
- 校验层、代理、容器与系统 resolver 的缓存策略不同。
- 相同域名在短时间内返回公网地址和 `127.0.0.1`/RFC1918 地址。

## 最小证据

- 分别记录校验时和连接时的解析结果、TTL、resolver 与时间。
- 证明实际 TCP 连接命中目标内网地址，而不只是 DNS 服务端日志变化。
- 排除客户端固定 IP、连接池、代理 DNS 缓存和应用层二次校验。

## 解法骨架

1. 读清 URL 校验与真实请求之间是否共享已验证 IP。
2. 使用唯一子域控制每轮状态，先返回可通过校验的公网地址，再切到内网地址。
3. 调整 TTL 和服务端状态机，避免本地/递归缓存吞掉第二次解析。
4. 用低副作用内网端点验证命中，再构造完整 SSRF/控制面请求。
5. 区分外层应用状态和内层目标状态，并为每一步使用新的解析名称。

## 关键变体

- DNS rebinding：同一名称按查询次数或时间切换地址。
- Split-horizon：不同 resolver/网络视角返回不同答案。
- Cache differential：校验层和连接层缓存生命周期不同。
- Host/SNI differential：IP 固定后，虚拟主机或 TLS 身份仍可能不一致。

## 常见陷阱

- 继续扩充 IP 字符串黑名单，却没有修复二次解析。
- 所有请求复用一个域名，递归缓存导致状态机失效。
- 只看挑战服务返回 `200`，没有检查目标端点的内层状态。
- 将 URL parser 差异与 DNS 时序混为一谈；两者的最小复现不同。

## 关联技巧

- [url-parser-wrapper-and-ssrf-filter-differential.md](url-parser-wrapper-and-ssrf-filter-differential.md)
- [dns-record-enumeration-and-zone-transfer.md](dns-record-enumeration-and-zone-transfer.md)
- [dns-tunnel-and-label-reassembly.md](dns-tunnel-and-label-reassembly.md)
- [web-and-dns.md](web-and-dns.md)

## 原始资料

- [dns.md](../raw/pentest/dns.md)
- [parser-wrapper-and-legacy-ssrf-tricks.md](../raw/web/parser-wrapper-and-legacy-ssrf-tricks.md)
- [SUCTF2026-uriWP](../raw/web/SUCTF2026-uriWP.md)
