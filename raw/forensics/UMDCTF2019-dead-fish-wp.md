# UMDCTF 2019 - Dead Fish

## 题目简述

附件是一份包含大量 TCP 流量的 PCAP。题面反复强调“握手”，提示应关注 TCP 三次握手，而不是在应用层载荷中搜索 flag。

## 解题过程

先筛出只设置 SYN、尚未设置 ACK 的初始握手包：

```bash
tshark -r suspicious.pcap \
  -Y 'tcp.flags.syn == 1 && tcp.flags.ack == 0' \
  -T fields -e frame.number -e ip.src -e tcp.srcport -e tcp.seq_raw
```

正常主机生成的初始序列号看起来接近随机数；来源 `192.168.0.1`、源端口 `1337` 的一组包却把序列号限制在可打印 ASCII 范围，例如 `85, 77, 68, 67, 84, 70`，对应 `UMDCTF`。按抓包顺序提取这一组序列号并转成字符：

```python
from scapy.all import IP, TCP, rdpcap

result = []
for packet in rdpcap("suspicious.pcap"):
    if IP not in packet or TCP not in packet:
        continue
    tcp = packet[TCP]
    if packet[IP].src == "192.168.0.1" and tcp.sport == 1337:
        if tcp.flags & 0x02 and not tcp.flags & 0x10:
            result.append(chr(tcp.seq))

print("".join(result))
```

得到：

```text
UMDCTF-{Th3_Tru3_3XF1L_Pr0Tocol}
```

其 SHA-256 与官方摘要一致。

## 方法总结

这是利用 TCP 初始序列号进行隐蔽传输。分析网络流量时，除了包体，还应检查序列号、端口、标志位、时间间隔等元数据。先用协议语义和异常分布缩小范围，再按帧顺序恢复字符，可以避免把正常的随机序列号误当成编码数据。
