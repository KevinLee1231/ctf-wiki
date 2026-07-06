---
type: family
tags: [misc, family, dns, rebinding, tunneling, enumeration]
skills: [ctf-misc, ctf-web, ctf-forensics, ctf-osint]
raw:
  - ../raw/misc/dns.md
updated: 2026-06-12
---

# DNS Exploitation Techniques

## 作用边界

本页是 DNS 协议利用与解析链 family，覆盖 ECS、DNSSEC NSEC walking、IXFR/AXFR 类 zone 枚举、DNS rebinding、DNS tunneling/exfiltration、round-robin 枚举、迷宫式解析、SPF/TXT 链和少量网络协议边界题。

它不并入 OSINT 的 [web-and-dns.md](web-and-dns.md)：OSINT 页关注公开情报查询，本页关注 DNS 协议行为、解析器差异、主动查询、隧道和 Web/内网 pivot。

## 识别信号

- 题面给出域名、权威 NS、SPF/TXT、DNSSEC、特殊 resolver、可控子域或要求从解析结果走迷宫。
- 查询结果随 ECS、源地址、递归/权威服务器、TTL、记录类型或时间变化。
- Web 题中 SSRF/浏览器/后端会解析攻击者控制域名，可能触发 rebinding。
- PCAP 或日志中出现大量 TXT/NULL/CNAME/子域标签，像隧道或外带。

## 最小证据

- 明确要查询的域、resolver、记录类型和权威服务器；不要只用默认系统 DNS。
- 对变化型记录，记录 TTL、查询时间、源地址和递归/权威差异。
- 对 rebinding，证明目标组件会重复解析，且访问控制用解析前后的不同结果。
- 对 tunneling，先还原方向、标签编码、分片顺序和是否压缩/加密。

## 路由表

| 证据 | 先验证 | 下一跳 |
|---|---|---|
| ECS 影响结果 | resolver 是否传递 ECS，伪造网段是否改变 A/TXT | 枚举目标地理/网段视角 |
| DNSSEC NSEC/NSEC3 | zone 是否可 walking，NSEC3 是否可字典恢复 | 枚举隐藏主机名 |
| IXFR/AXFR/zone transfer | 权威 NS 是否允许 transfer 或增量泄露 | 拉取 zone 并查敏感记录 |
| DNS rebinding | 目标是否重复解析且不固定 IP | 转 [polyglot-url-tricks-and-ssrf-leaks.md](polyglot-url-tricks-and-ssrf-leaks.md) 或 SSRF 页 |
| DNS tunneling/exfil | 查询标签高熵、分片、有序号或 base 编码 | 转 [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md) |
| round-robin 枚举 | 同一记录多次查询返回不同 A/AAAA | 多 resolver、多次采样去重 |
| DNS maze | 每次记录指向下一域名或下一记录类型 | 脚本化遍历，避免缓存干扰 |
| SPF/TXT 链 | `include:`、`redirect=`、TXT 拼接暴露 flag | 递归展开并处理字符串拼接 |
| TCP Fast Open / SYN payload | 题目本质是网络协议边界，不是 DNS | 转网络/forensics 或 pwn primitive 判断 |

## 合并与拆分结论

- 保留为 family：DNS 题经常在 misc、web、forensics、osint 之间切换，本页提供协议层 pivot。
- 不合并进 `misc-cross-category-triage-family.md`：总入口只决定是否进入 DNS，本页负责 DNS 内部路线。
- 不合并进 OSINT `web-and-dns.md`：公开查询和协议利用的证据、工具和失败状态不同。

## 常见误判

- 只查默认 resolver，漏掉权威服务器、ECS 或缓存差异。
- DNSSEC walking 时没有区分 NSEC 和 NSEC3，导致错误期待明文枚举。
- Rebinding 只在浏览器测试成功，没有证明目标后端真的重复解析。
- 隧道流量没有按 query/response 方向和标签顺序重组。

## 关联页面

- [misc-cross-category-triage-family.md](misc-cross-category-triage-family.md)
- [web-and-dns.md](web-and-dns.md)
- [polyglot-url-tricks-and-ssrf-leaks.md](polyglot-url-tricks-and-ssrf-leaks.md)
- [parser-wrapper-and-legacy-ssrf-tricks.md](parser-wrapper-and-legacy-ssrf-tricks.md)
- [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md)
- [misc-tooling.md](misc-tooling.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [D3CTF2022-ohhhh-spf-wp](../raw/misc/D3CTF2022-ohhhh-spf-wp.md) | SPF/DNS 记录是主线，先查询 TXT/SPF 解析链并确认域名授权语义。 |

## 原始资料

- [dns.md](../raw/misc/dns.md)
