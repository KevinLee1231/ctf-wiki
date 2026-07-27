---
type: technique
tags: [osint, pentest, forensics, dns, zone-transfer, tunnel]
skills: [ctf-osint, ctf-pentest, ctf-forensics]
raw:
  - ../raw/pentest/dns.md
  - ../raw/forensics/network-covert-auth-and-reassembly.md
updated: 2026-07-27
---

# DNS Zone-Transfer, Tunnel and Resolution Analysis

## 适用场景

DNS 记录、权威区配置、解析时序或 label 流量泄露主机、凭据和隐藏数据；需要区分资产枚举、zone transfer、rebinding 与 DNS tunnel 重组。

## 识别信号

- NS/SOA/TXT/SPF/SRV/CAA/CT 暴露子域或服务。
- 权威服务器允许 AXFR/IXFR，或 split-horizon 返回差异。
- PCAP 中 query name 高熵、序号化、固定长度且响应节奏异常。

## 最小证据

- 记录 resolver、authoritative NS、查询类型、TTL 和原始响应。
- Zone transfer 需保存完整区域并确认权威来源。
- Tunnel 需证明 label 顺序、编码和重组结果。

## 解法骨架

1. 从 NS/SOA 开始枚举记录、委派和权威边界。
2. 测试 AXFR/IXFR、wildcard、DNSSEC/NSEC 与 split-horizon。
3. 流量场景按 client/domain/序号排序 label，去除 framing 后解码。
4. 若涉及 rebinding，分别记录校验时与连接时解析结果和 TTL。

## 关键变体

- Zone transfer/record enumeration。
- DNS tunnel/covert label reassembly。
- Rebinding、cache/TTL 与解析链差异。

## 常见陷阱

- 用递归 resolver 的缓存结果代替权威证据。
- 暴力子域枚举未处理 wildcard。
- Tunnel 直接按抓包顺序拼接，忽略重传和序号。

## 关联技巧

- [dns.md](dns.md)
- [web-and-dns.md](web-and-dns.md)
- [protocol-stream-reassembly-and-credential-extraction.md](protocol-stream-reassembly-and-credential-extraction.md)

## 原始资料

- [dns.md](../raw/pentest/dns.md)
- [network-covert-auth-and-reassembly.md](../raw/forensics/network-covert-auth-and-reassembly.md)
