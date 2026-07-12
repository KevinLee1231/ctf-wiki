---
type: family
tags: [forensics, family, pcap, covert-channel, auth, reassembly]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/network-covert-auth-and-reassembly.md
  - ../raw/forensics/WMCTF2025-voice-hacker-wp.md
updated: 2026-06-12
---

# Network Covert Channels, Auth and Reassembly

## 作用边界

本页是网络取证二级 family，覆盖 PCAP 中的时序/flag/DNS/ICMP covert channel、NTLMv2/MS-SNTP/RADIUS/RDP 凭据、dnscat2/多层 TCP 重组、Brotli/ZIP/XOR 组合、SMB/LSARPC 线索和 RTP 音频恢复。

它不替代 [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md) 的首轮入口；本页负责在确认是网络证据后，判断是 covert encoding、凭据恢复、协议重组、内部认证绕过还是多层文件恢复。

## 识别信号

- 附件是 pcap/pcapng，或磁盘/内存中导出的网络流量。
- Wireshark 正常 Follow Stream 不直接给 flag，需要看时间间隔、TCP flags、DNS name、ICMP payload、RTP、SMB/RPC、NTP/RADIUS/RDP 或压缩/加密层。
- 证据可能是可破解 hash、会话密钥、音频认证样本、重组文件、covert bits 或内部协议调用。

## 最小证据

- 固定过滤条件：接口、IP、端口、协议、stream id、时间窗口和方向。
- 对 covert channel，记录 bit 映射、阈值、排序、端序和是否需要补首位。
- 对认证协议，记录 hash 格式、salt、key id、证书/私钥、shared secret 或导出音频。
- 对重组文件，保留 raw stream、解压/异或/合并顺序和每层校验。

## 首轮路由

| 证据形态 | 首轮判断 | 下一跳 |
|---|---|---|
| packet interval、TCP flag、DNS label、ICMP payload | 先提取字段序列和 bit 映射，再转编码/统计验证 | [encodings-qr-and-esolangs.md](encodings-qr-and-esolangs.md) |
| NTLMv2、MS-SNTP、RADIUS、RDP TLS | 先导出 hash/key/cert，再用合适格式破解或导入 Wireshark 解密 | [forensics-tooling.md](forensics-tooling.md) |
| dnscat2、fake TLS、multi-layer XOR+ZIP | 先按 stream/session 重组，再逐层解压、异或和 merge | [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md) |
| SMB/LSARPC、RID、Timeroasting | 先确认域/账号/RID 语义，再判断是否转 pentest 凭据链 | [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md) |
| RTP/UDP 音频流 | 先 Decode As RTP 并导出 wav，再判断是音频取证还是认证绕过样本 | [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md) |
| PDF/文件对象从网络流恢复 | 先重组对象和 xref，再转文件格式页面 | [pdf-png-gif-and-text-stego.md](pdf-png-gif-and-text-stego.md) |
| RC4/自定义加密流量 | 先定位 key/keystream，再转 crypto 或 malware 协议恢复 | [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md), [malware-c2-session-key-and-protocol-recovery.md](malware-c2-session-key-and-protocol-recovery.md) |

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2025-voice-hacker-wp](../raw/forensics/WMCTF2025-voice-hacker-wp.md) | PCAP 中一批 UDP 包可按 RTP 解码导出目标音频；后续不是解协议 flag，而是把音频样本用于绕过 `/api/authenticate` 的 wav 特征相似度。 |

## 合并与拆分结论

本页应保留为 family。网络时序 covert、凭据恢复、协议重组和音频认证绕过的第一步证据不同；但它们都发生在 PCAP/网络流量层，适合作为 PCAP 首轮之后的二级入口。

## 常见陷阱

- 没固定 stream/filter，导致不同连接的数据混在一起。
- Covert channel 不记录排序和端序，复现时 bit 串变化。
- 认证协议只看明文字段，没导出 hashcat/john 所需格式。
- RDP/RADIUS 这类协议没导入 key/secret，误以为无法解密。
- RTP/音频流没有 Decode As，错过可直接导出的 wav。

## 关联技巧

- [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)
- [cross-domain-forensics-technique-map.md](cross-domain-forensics-technique-map.md)
- [audio-frequency-and-archive-stego.md](audio-frequency-and-archive-stego.md)
- [file-triage-archives-and-one-liners.md](file-triage-archives-and-one-liners.md)
- [rc4-lfsr-and-keystream-reuse.md](rc4-lfsr-and-keystream-reuse.md)
- [forensics-tooling.md](forensics-tooling.md)

## 原始资料

- [network-covert-auth-and-reassembly.md](../raw/forensics/network-covert-auth-and-reassembly.md)
- [WMCTF2025-voice-hacker-wp](../raw/forensics/WMCTF2025-voice-hacker-wp.md)
