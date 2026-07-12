---
type: technique
tags: [pentest, network, bgp, rpki, technique]
skills: [ctf-pentest]
raw:
  - ../raw/pentest/D3CTF2025-d3rpki-wp.md
updated: 2026-07-06
---

# BGP RPKI Route Hijack

## 适用场景

题目给出 BGP / RPKI / AS / CIDR 配置，要求访问某个看似由其他自治域持有的地址，或者在路由策略里改变目标前缀的可达路径。

## 识别信号

- 配置中出现 `protocol bgp`、peer、AS number、ROA、RPKI、CIDR 或 `route ... reject`。
- 目标服务不在本机网段，但题目允许配置或伪装 BGP peer。
- RPKI 校验只确认某个前缀是否属于某个 AS，未同时验证宣告路径、peer 可信度或更具体前缀来源。

## 最小证据

- 已确认目标 IP / CIDR、合法 origin AS、当前路由表和可建立的 peer 关系。
- 能观察到路由导入、导出或 reject 规则如何处理目标前缀。
- 能证明宣告更具体路由或伪造 peer 后，目标流量会经由攻击者可控路径。

## 解法骨架

1. 提取目标 IP、合法 AS、已有 BGP peer 和 RPKI/ROA 约束。
2. 判断校验的是 origin 还是完整路径；如果只校验 origin，寻找可伪装的 peer 或更具体前缀宣告点。
3. 宣告能覆盖目标 IP 的路由，优先使用更具体前缀压过 reject 或默认路由。
4. 验证路由表和服务连通性，再从被劫持路径访问 flag 服务。

## 关键变体

| 变体 | 判断重点 |
|---|---|
| origin-only RPKI | ROA 校验通过不代表宣告路径可信，可伪造 peer 或利用缺失的 AS-SET 校验。 |
| 更具体前缀覆盖 | `/32` 或更长前缀可能压过宽前缀 reject，先确认路由优先级。 |
| 本地 lab BGP | 关注 Bird/FRR 配置、import/export filter 和 route table，而不是公网真实可达性。 |

## 常见陷阱

- 把 RPKI 当成完整的 BGP 路径认证，忽略它通常只覆盖 origin 授权。
- 只看 CIDR 归属，不检查是否能作为 peer 宣告更具体路由。
- 修改路由前不保存原始配置和路由表，导致失败后无法判断是哪条 filter 拦截。

## 关联技巧

- [web-and-dns.md](web-and-dns.md)
- [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md)
- [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md)

## 原始资料

- [D3CTF2025-d3rpki-wp.md](../raw/pentest/D3CTF2025-d3rpki-wp.md)
