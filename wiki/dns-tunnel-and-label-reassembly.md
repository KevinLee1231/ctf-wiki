---
type: technique
tags: [forensics, malware, dns, tunnel, covert-channel, reassembly]
skills: [ctf-forensics, ctf-malware]
raw:
  - ../raw/forensics/network-covert-auth-and-reassembly.md
  - ../raw/forensics/pcap-protocol-credential-recovery.md
  - ../raw/malware/c2-and-protocols.md
updated: 2026-07-28
---

# DNS Tunnel and Label Reassembly

## 适用场景

PCAP 或恶意流量把数据编码进 DNS query name、TXT/CNAME 响应或查询时序；需要去重、排序、去 framing 并解码，恢复文件、命令、凭据或 C2 数据。

## 识别信号

- 子域 label 高熵、长度固定、带序号或明显 Base32/Base64/hex 字符集。
- 同一 client/domain 高频查询，响应类型与正常业务不匹配。
- query/response 中存在 session ID、chunk index、方向位、校验和或重传。

## 最小证据

- 固定 client、server、主域、时间窗和查询类型，避免混入正常 DNS。
- 证明 chunk 的排序、重传去重、编码和方向判定规则。
- 重组结果通过文件 magic、长度、hash、协议语法或正向解码验证。

## 解法骨架

1. 过滤目标 client/domain，导出原始 query/response 与时间戳。
2. 拆分主域、session、sequence、payload 和校验字段。
3. 按协议序号而非抓包顺序重排，去重并标记缺包。
4. 逐层执行 Base32/Base64/hex、压缩、XOR 或加密恢复。
5. 按文件头、命令语法或双向会话校验最终对象。

## 常见陷阱

- 直接按抓包顺序拼接，忽略重传、并发 session 和乱序。
- 把 DNS 名称大小写规范化导致大小写信道丢失。
- 只分析 query，不检查 TXT/CNAME 响应方向。
- 看到 DNS 就先做 AXFR；流量重组与资产枚举是不同问题。

## 关联技巧

- [protocol-stream-reassembly-and-credential-extraction.md](protocol-stream-reassembly-and-credential-extraction.md)
- [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md)
- [dns-record-enumeration-and-zone-transfer.md](dns-record-enumeration-and-zone-transfer.md)
- [malware-c2-session-key-and-protocol-recovery.md](malware-c2-session-key-and-protocol-recovery.md)

## 原始资料

- [network-covert-auth-and-reassembly.md](../raw/forensics/network-covert-auth-and-reassembly.md)
- [pcap-protocol-credential-recovery.md](../raw/forensics/pcap-protocol-credential-recovery.md)
- [c2-and-protocols.md](../raw/malware/c2-and-protocols.md)
