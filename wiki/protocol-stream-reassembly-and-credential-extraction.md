---
type: technique
tags: [forensics, pcap, reassembly, credentials, protocol]
skills: [ctf-forensics]
raw:
  - ../raw/forensics/network-covert-auth-and-reassembly.md
  - ../raw/forensics/pcap-protocol-credential-recovery.md
  - ../raw/forensics/UMDCTF2017-leetfiltration-wp.md
  - ../raw/forensics/WMCTF2023-oversharing-wp.md
updated: 2026-07-28
---

# Protocol Stream Reassembly and Credential Extraction

## 适用场景

PCAP/日志中应用对象、凭据、token 或文件被 TCP 分段、重传、乱序、分块编码或封装在 DNS/HTTP/自定义协议中，需先按会话重组再解释。

## 识别信号

- 单包看不到完整对象，但同一 5-tuple 持续交换分段数据。
- HTTP chunked、WebSocket、TLS key log、DNS label 或自定义 length field。
- 登录/认证消息、cookie、Authorization 或挑战响应可关联。

## 最小证据

- 明确会话键、方向、序号、重传和缺包情况。
- Reassembled bytes 可由协议 parser/长度/校验验证。
- 凭据来源与后续成功会话有时间和端点关联。

## 解法骨架

1. 按协议/会话分组，处理重传、乱序和方向。
2. 去除传输 framing，再处理 chunk/compression/encoding/encryption。
3. 提取对象、凭据或 challenge-response 并保留包号范围。
4. 与后续请求、登录结果或文件 hash 交叉验证。

## 关键变体

- TCP/HTTP/WebSocket object reassembly。
- DNS tunnel/label 编码。
- TLS 会话有 key log/弱密钥时解密后再重组。

## 常见陷阱

- 直接拼 payload，忽略重传和序号。
- 将客户端/服务端方向混合。
- 只发现字符串，没有证明属于成功认证会话。

## 关联技巧

- [pcap-protocol-credential-recovery-family.md](pcap-protocol-credential-recovery-family.md)
- [network-covert-auth-and-reassembly.md](network-covert-auth-and-reassembly.md)
- [dns-tunnel-and-label-reassembly.md](dns-tunnel-and-label-reassembly.md)

## 原始资料

- [network-covert-auth-and-reassembly.md](../raw/forensics/network-covert-auth-and-reassembly.md)
- [pcap-protocol-credential-recovery.md](../raw/forensics/pcap-protocol-credential-recovery.md)
- [UMDCTF2017-leetfiltration-wp](../raw/forensics/UMDCTF2017-leetfiltration-wp.md)
- [WMCTF2023-oversharing-wp](../raw/forensics/WMCTF2023-oversharing-wp.md)
