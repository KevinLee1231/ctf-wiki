---
type: technique
tags: [osint, dns, whois, ct, archive, pivot]
skills: [ctf-osint]
raw:
  - ../raw/osint/web-and-dns.md
  - ../raw/osint/social-media.md
updated: 2026-07-27
---

# Public-Record, DNS/WHOIS and Archive Pivoting

## 适用场景

从域名、组织、邮箱、文档或旧 URL 出发，需要在 DNS/WHOIS/CT、搜索缓存、Wayback、公开仓库和历史页面间建立可验证 pivot 链。

## 识别信号

- 当前站点已删除/重定向，但域名、证书、子域或旧路径仍可查。
- 邮箱、组织名、tracking id、analytics 或文档 metadata 可跨来源关联。
- 历史 DNS/WHOIS/仓库 commit 暴露旧资产或身份。

## 最小证据

- 每个 pivot 保存来源 URL、查询时间、原始字段和截图/文本。
- 同一实体至少有两个独立稳定标识关联。
- 明确历史事实与当前状态，不把过期记录当现状。

## 解法骨架

1. 规范化初始 indicator：域名、邮箱、账号、组织和时间范围。
2. 查询 DNS/CT/WHOIS 与 archive/search cache，扩展子域、旧 URL 和联系人。
3. 将新 indicator 回查公开仓库、文档 metadata 和社交资料。
4. 构建带时间和来源的证据图，只保留可重复验证的边。

## 关键变体

- DNS/CT/WHOIS 基础设施 pivot。
- Wayback/search cache 历史内容恢复。
- Public repo/document metadata identity pivot。

## 常见陷阱

- 只凭相同昵称/头像认定同一实体。
- 不记录查询时间，混淆历史与当前归属。
- 引用搜索摘要而未打开原始来源。

## 关联技巧

- [web-and-dns.md](web-and-dns.md)
- [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md)
- [dns-zone-transfer-tunnel-and-resolution-analysis.md](dns-zone-transfer-tunnel-and-resolution-analysis.md)

## 原始资料

- [web-and-dns.md](../raw/osint/web-and-dns.md)
- [social-media.md](../raw/osint/social-media.md)
