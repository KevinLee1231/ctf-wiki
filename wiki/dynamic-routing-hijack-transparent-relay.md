---
type: technique
tags: [pentest, network, ospf, route-hijack, transparent-relay, technique]
skills: [ctf-pentest]
raw:
  - ../raw/pentest/D3CTF2024-OSPF-Enhanced-wp.md
updated: 2026-07-27
---

# Dynamic Routing Hijack and Transparent Relay

## 适用场景

攻击者能加入 OSPF 或其它内部动态路由域，并可宣告目标主机的更精确路由。目标不是让服务改连攻击者，而是在保持客户端和真实服务器会话语义的前提下把流量引经攻击者，进行观察、关联或改写。

## 识别信号

- VPN/实验网提供路由邻居、OSPF/Bird/FRR 配置或允许发送路由公告。
- 目标 IP 已有较宽网段路由，而攻击者能宣告 `/32` 等更精确前缀。
- 单纯宣告后服务超时或形成黑洞，说明仍需转发到真实服务器。
- 攻击目标依赖旁路可见信息，例如 TLS key、TCP ACK 差分、载荷长度或签名对应关系。

## 最小证据

- 保存原始路由表、邻居状态、目标前缀和真实下一跳。
- 证明更精确路由被其它节点接受，目标流量确实到达攻击接口。
- 设计回送路径，确认转发后的源/目的地址、二层邻居和 TCP 状态仍成立。
- 对侧信道关联给出唯一性条件，例如每条明文长度是否唯一。

## 解法骨架

1. 通过 traceroute、配置和抓包恢复拓扑、目标前缀与原下一跳。
2. 加入动态路由域，先宣告测试前缀验证邻居和过滤策略。
3. 宣告覆盖目标的更精确路由，将流量导向攻击机。
4. 使用二层转发、策略路由或透明桥接把报文继续送往真实服务器，避免黑洞和本机路由回环。
5. 在透明路径上提取目标侧信道或实施最小改写，并用端到端会话成功作为验证。

## 关键变体

- OSPF 依赖内部邻接和度量，BGP/RPKI 的 origin/AS path 约束属于不同路由模型。
- 只需被动观察时优先保持包内容和序号不变；主动改写还需处理校验和、长度和 TCP 状态。
- Linux 本机可能因 local route 或 rp_filter 吞包，需分别验证路由层与二层转发。
- ACK 序号差可泄露载荷长度，但只有长度映射唯一时才能稳定对应业务消息。

## 常见陷阱

- 把路由劫持等同于中间人，未建立到真实服务器的回送链。
- 宣告前不保存路由表，失败后无法区分邻居、过滤和转发问题。
- 用 NAT 改变连接语义，破坏本应透明观察的 TCP 会话。
- 看到长度侧信道就直接映射消息，未验证不同明文是否同长。
- 将 OSPF 内网劫持与 BGP/RPKI origin 绕过混成同一个验证模型。

## 关联技巧

- [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md)
- [bgp-rpki-route-hijack.md](bgp-rpki-route-hijack.md)
- [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)
- [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md)
- [pentest-tooling.md](pentest-tooling.md)

## 原始资料

- [D3CTF2024-OSPF-Enhanced-wp.md](../raw/pentest/D3CTF2024-OSPF-Enhanced-wp.md)
