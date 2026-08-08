# MiniLCTF2020 - MITM_0

## 题目简述

附件是一份局域网中间人攻击 PCAP，题目要求找出 bad guy 的 IP 并对它做 Base64。分析重点是联合 ARP、Ethernet、IPv4 和 HTTP/TLS 会话，而不是只按单一协议统计流量。

## 解题过程

先用下面的显示过滤器检查 ARP：

```text
arp
```

流量中反复出现 `192.168.1.1 is at 00:0c:29:3b:d3:41`，同时局域网主机 `192.168.1.152` 与大量公网 TLS 端点通信，并与少量明文 HTTP 流量形成转发关系。继续过滤：

```text
http && ip.addr == 192.168.1.152
```

可以看到该地址出现在全部关键 HTTP 会话中，符合中间人转发节点的行为。原题接受的 bad guy IP 为：

```text
192.168.1.152
```

按题面要求只对 ASCII IP 字符串做标准 Base64：

```python
import base64
print(base64.b64encode(b'192.168.1.152').decode())
```

结果是：

```text
MTkyLjE2OC4xLjE1Mg==
```

提交格式为 `minil{MTkyLjE2OC4xLjE1Mg==}`。PCAP 中 IP 与 MAC 的异常对应关系还会在 `MITM_2` 中进一步解释，因此不能把某一条 ARP 应答孤立地当作最终证明。

## 方法总结

识别 MITM 节点应使用多层证据：ARP 声明、源/目的 MAC、IP 会话方向、协议降级和转发流量分布。Base64 的输入是题面指定的原始 IP 文本，不应附加换行，否则编码结果会不同。
