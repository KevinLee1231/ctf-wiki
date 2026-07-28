---
type: technique
tags: [osint, pentest, dns, records, zone-transfer]
skills: [ctf-osint, ctf-pentest]
raw:
  - ../raw/pentest/dns.md
  - ../raw/osint/web-and-dns.md
updated: 2026-07-28
---

# DNS Record Enumeration and Zone Transfer

## 适用场景

公开 DNS 记录、权威区配置或不当开放的 AXFR/IXFR 能暴露子域、主机角色、邮件与服务边界；目标是扩展资产和关系图，而不是分析抓包中的隐蔽载荷。

## 识别信号

- 已知根域、NS、SOA、MX、TXT、SPF、SRV、CAA 或少量子域线索。
- 权威服务器可能允许 AXFR/IXFR，或 wildcard/split-horizon 影响枚举结果。
- CT、DNSSEC NSEC/NSEC3、被动 DNS 与实时权威响应可互相校验。

## 最小证据

- 记录查询时间、查询类型、resolver 与 authoritative NS。
- Zone transfer 结果必须来自权威服务器，并保留完整响应。
- 对 wildcard、缓存和历史记录分别标注，不能把它们当当前主机存在证明。

## 解法骨架

1. 从 NS/SOA 确认权威边界和委派关系。
2. 查询 MX/TXT/SRV/CAA 等高信息量记录，再做受控子域枚举。
3. 对每台权威 NS 测试 AXFR/IXFR，并识别 wildcard。
4. 结合 CT、被动 DNS 和 Web 证书/响应验证主机用途。
5. 将确定的记录作为下一步服务枚举或实体关联入口。

## 常见陷阱

- 用递归 resolver 的缓存结果代替权威证据。
- 暴力枚举未处理 wildcard，产生大量伪子域。
- 把历史 DNS/CT 记录直接视为当前可达资产。
- 将高熵 DNS label 误归本页；PCAP 重组应转 DNS tunnel 技巧。

## 关联技巧

- [dns-tunnel-and-label-reassembly.md](dns-tunnel-and-label-reassembly.md)
- [dns-resolution-rebinding-and-cache-differentials.md](dns-resolution-rebinding-and-cache-differentials.md)
- [public-record-dns-whois-and-archive-pivoting.md](public-record-dns-whois-and-archive-pivoting.md)
- [dns.md](dns.md)

## 原始资料

- [dns.md](../raw/pentest/dns.md)
- [web-and-dns.md](../raw/osint/web-and-dns.md)
