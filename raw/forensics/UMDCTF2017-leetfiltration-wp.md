# leetfiltration

## 题目简述

`capture.cap` 包含 734 个本机回环 TCP 数据包。流量由源端口 33317 发往端口 5000，应用层几乎只出现字节 `13` 和 `37`。题名中的 “leet” 对应十六进制写法 `0x1337`，说明这两种符号被用作二进制隐蔽信道。

## 解题过程

先按抓包顺序拼接客户端到服务端的 TCP 负载。发送端用单字节 `0x13` 表示 0，用双字节 `0x13 0x37` 表示 1；TCP 可能合并多个发送操作，因此不能把“每个包”直接当成一个 bit，必须在重组后的字节流上解析：

```python
from scapy.all import IP, Raw, TCP, rdpcap

stream = b"".join(
    bytes(packet[Raw].load)
    for packet in rdpcap("capture.cap")
    if packet.haslayer(IP)
    and packet.haslayer(TCP)
    and packet[TCP].dport == 5000
    and packet.haslayer(Raw)
)

bits = []
offset = 0
while offset < len(stream):
    assert stream[offset] == 0x13
    is_one = offset + 1 < len(stream) and stream[offset + 1] == 0x37
    bits.append("1" if is_one else "0")
    offset += 2 if is_one else 1

message = bytes(
    int("".join(bits[index:index + 8]), 2)
    for index in range(0, len(bits), 8)
)
print(message.decode())
```

368 个 bit 正好组成 46 字节，输出：

```text
=== UMDCTF-{exfiltration_thr0ugh_patt3rns} ===
```

取中间的标准 flag，其 SHA-256 与 README 中的 `1f5205d2081b656f027ada55bf919012886fa5e7d3d6256215177ff5268bc2d9` 一致。

## 方法总结

网络隐蔽信道的逻辑符号不一定与数据包一一对应。TCP 是字节流协议，分段和合并都可能改变包边界；可靠做法是先按方向重组负载，再根据自描述的前缀规则解析 `0x13` 与 `0x1337`，最后按 8 bit 大端恢复字节。
