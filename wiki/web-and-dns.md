---
type: family
tags: [osint, family, public-web, dns, archive, public-index]
skills: [ctf-osint]
raw:
  - ../raw/osint/web-and-dns.md
updated: 2026-07-06
---

# Web and DNS OSINT

## 作用边界

本页是 Web/DNS 公开来源 family，用于从搜索引擎、公开文档、DNS/TXT/SPF、WHOIS/ASN、Wayback、Shodan、GitHub、Telegram、Tor relay、公开仓库、目录泄露和服务 banner 中定位信息。它关注公开证据收集，不处理对目标服务的技术性漏洞利用。

如果需要登录、利用 Web 漏洞或内网打点，应转 [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md) 或 Pentest；如果线索已经变成账号身份链，转 [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md)。

## 识别信号

- 题目给出域名、URL、公开文档、仓库、用户名、邮箱、服务指纹、SSH/TLS fingerprint、TXT/SPF 记录、历史页面或搜索关键词。
- 目标是找到公开暴露的 flag、历史版本、真实作者、关联域名、隐藏目录、公开服务位置或被索引的敏感内容。
- 证据可通过公开 URL/API 复查，且不需要主动攻击目标应用。

## 最小证据

- 保存查询语句、URL、时间、结果页面和关键字段；搜索结果本身会漂移。
- 对域名/DNS，记录 resolver、记录类型、TTL、历史记录来源和是否存在 wildcard。
- 对历史页面，记录 archive URL、快照时间和当前页面差异。
- 对 Shodan/指纹，记录 fingerprint 来源、匹配查询和目标服务最小验证方式。

## 首轮路由

| 证据形态 | 先做什么 | 下一跳 |
|---|---|---|
| 搜索引擎/公开文档 | 用精确关键词、引号、`site:`、文件类型和片段组合；保存结果 URL 而不是只截图。 | [osint-tooling.md](osint-tooling.md) |
| DNS/TXT/SPF/WHOIS/ASN | 枚举 A/AAAA/CNAME/MX/TXT/NS/SOA，检查 SPF include 链、历史 WHOIS、ASN 和共享基础设施。 | [dns.md](dns.md) |
| Wayback/历史页面 | 对比不同时间快照，寻找删除的 flag、旧路径、源码、备份文件和迁移线索。 | [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md) |
| GitHub/Git commits/公开仓库 | 查 commit author、issue、PR、README、历史 blob、邮箱、用户名和外链。 | [linux-git-browser-and-container-forensics.md](linux-git-browser-and-container-forensics.md) |
| Shodan/SSH/TLS/banner | 先获取本地 fingerprint 或 banner，再公开搜索匹配；假 banner 要用多工具确认。 | [osint-tooling.md](osint-tooling.md) |
| `.DS_Store`、公开目录、静态文件泄露 | 先确认是否公开索引证据；若需要下载/恢复对象，转文件取证。 | [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md) |
| Telegram/Tor/FEC/平台公开 API | 先确认平台数据是否公开、稳定和可复查，再把结果并入身份链。 | [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md) |

## 合并与拆分结论

- 保留为 `family`：raw 覆盖搜索、DNS、历史快照、WHOIS、Shodan、GitHub、Telegram、Tor、公开目录和服务指纹，核心价值是公开来源二级分流。
- 不并入 [geolocation-and-media.md](geolocation-and-media.md)：本页的起点是域名/公开网页/服务指纹，地理页的起点是图片/坐标/媒体内容。
- 不拆 DNS、Wayback、Shodan 小页：当前 OSINT 页面数量少，保留三大 family 入口更清晰。

## 常见陷阱

- 搜索命中后不保存原始 URL 和时间，后续无法复查。
- DNS 只查 A 记录，不看 TXT/SPF/NS/MX/历史 WHOIS。
- Wayback 只看最近快照，遗漏早期泄露和页面迁移。
- Shodan 指纹未确认来源，拿错误 fingerprint 搜索导致假阳性。
- 公开目录泄露已经转为文件恢复时仍按 OSINT 搜索，错过 `.git`/archive 取证路线。

## 关联技巧

- [geolocation-and-media.md](geolocation-and-media.md)
- [osint-account-public-media-correlation.md](osint-account-public-media-correlation.md)
- [dns.md](dns.md)
- [linux-git-browser-and-container-forensics.md](linux-git-browser-and-container-forensics.md)
- [filesystem-archive-recovery-and-repair.md](filesystem-archive-recovery-and-repair.md)
- [web-first-pass-triage-and-chain-patterns.md](web-first-pass-triage-and-chain-patterns.md)
- [osint-tooling.md](osint-tooling.md)

## 原始资料

- [web-and-dns.md](../raw/osint/web-and-dns.md)
