# BSidesAlgiers2025 - HBH

## 题目简述

`HBH` 指 IPv6 Hop-by-Hop 扩展头。题目把数据藏在 ICMPv6 Echo 报文的 Hop-by-Hop `PadN` 选项中，同时混入 SSH、HTTP、DNS 等无关流量。恢复链为：过滤指定 IPv6 源与扩展头、按 ICMPv6 序号排序、提取 `PadN`、逐字节异或 `0x42`，最后做 Base64 解码。

## 解题过程

先锁定源地址 `2001:db8:1234:5678::42`，并要求 IPv6 `Next Header` 为 `0`，即后面跟随 Hop-by-Hop 扩展头。序号决定分片顺序，不能按抓包中的偶然到达顺序直接拼接：

```bash
tshark -r chall.pcap \
  -Y "ipv6.src == 2001:db8:1234:5678::42 && ipv6.nxt == 0" \
  -T fields \
  -e icmpv6.echo.sequence_number \
  -e ipv6.opt.padn | \
sort -n | cut -f2 | cut -d, -f1 | tr -d ':' | tr -d '\n' | \
python3 -c 'import sys; d=bytes.fromhex(sys.stdin.read()); print(bytes(x^0x42 for x in d).decode())' | \
base64 -d
```

本地用 Scapy 独立复核时共得到 10 个有效分片。排序、异或后的中间值是：

```text
c2hlbGxtYXRlc3toMHBfYnlfaDBwXzBwdDEwbnNfaDFkM19zM2NyM3RzXzFuX3BsNDFuX3MxZ2h0fQ==
```

Base64 解码得到最终 flag：

`shellmates{h0p_by_h0p_0pt10ns_h1d3_s3cr3ts_1n_pl41n_s1ght}`

## 方法总结

- IPv6 隐蔽信道不一定使用应用层 payload；Hop-by-Hop、Destination Options、Flow Label 等低关注字段都值得检查。
- 逆向顺序必须是“按序号重排 → 去字段格式 → 异或 → Base64”，任一步颠倒都会破坏后续编码边界。
- 保留 Base64 中间值能把字段提取错误与最终解码错误分开，便于逐层验证。
