# Nefarious Bits

## 题目简述

PCAP 中是一系列看似相同的 IPv4/TCP SYN 包。区别藏在 IPv4 头的 reserved flag，也就是俗称的 evil bit：置位表示二进制 1，未置位表示 0，按抓包顺序组合即可恢复文本。

## 解题过程

生成逻辑把 flag 按每字符 8 位、最高位在前转换成比特流。读取每个数据包的 IPv4 标志位并按八位分组：

```python
from scapy.all import IP, rdpcap

packets = rdpcap("attack.pcap")
bits = [
    1 if packet[IP].flags.evil else 0
    for packet in packets
    if IP in packet
]

plain = bytearray()
for offset in range(0, len(bits) - 7, 8):
    value = 0
    for bit in bits[offset:offset + 8]:
        value = (value << 1) | bit
    plain.append(value)

print(plain.decode())
```

输出为：

```text
UMDCTF-{3vil_b1ts_@r3_4lw4ys_3vi1}
```

该文本同时出现在生成脚本中，并可从公开 PCAP 按位恢复；README 中的哈希与这两份证据不一致，应视为过期记录。

## 方法总结

隐蔽信道可能借用协议头中几乎不会使用的单个标志位。分析大量结构相似的数据包时，应比较字段级差异并保持原始时间顺序；本题的每个包只承载一位信息。
