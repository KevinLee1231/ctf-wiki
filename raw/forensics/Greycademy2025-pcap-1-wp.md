# PCAP 1

## 题目简述

附件记录了一个未加密登录服务的 HTTP 流量，目标是找到用户名为 `greyhats` 的登录尝试。凭据直接出现在 TCP 载荷中，因此核心是按字符串或 TCP stream 定位明文请求。

## 解题过程

在 Wireshark 中搜索 packet details 里的字符串 `greyhats`，或直接过滤 TCP 载荷：

```bash
tshark -r baby.pcap \
  -Y 'tcp contains "greyhats"' \
  -T fields -e frame.number -e tcp.stream -e tcp.payload
```

命中第 148 帧、`tcp.stream == 10`。将载荷按十六进制解码后得到：

```json
{"username": "greyhats", "password": "grey{ju57_f0110w_7h3_57234m}"}
```

因此 flag 为：

```text
grey{ju57_f0110w_7h3_57234m}
```

## 方法总结

处理明文认证 PCAP 时，先用已知用户名做字符串定位通常比逐流翻阅更快；命中后再固定 `tcp.stream` 复核完整上下文。HTTP 上的 JSON、表单字段和 Basic Auth 如果没有 TLS 保护，都可能直接泄露凭据。
