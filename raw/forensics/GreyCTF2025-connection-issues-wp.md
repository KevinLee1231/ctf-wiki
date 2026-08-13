# Connection Issues

## 题目简述

题目给出员工断网时捕获的 `chall.pcap`，要求找出断网原因并恢复隐藏信息。流量中出现大量冒充网关的 ARP Reply；攻击者还把 Base64 文本分片放入这些以太网帧的尾随字节中。

## 解题过程

先围绕异常网关 `192.168.100.1` 检查 ARP Reply。恶意帧统一使用源 MAC `bc:24:11:78:c8:64`，可用以下显示过滤器缩小范围：

```text
arp.opcode == 2 &&
arp.src.hw_mac == bc:24:11:78:c8:64 &&
arp.src.proto_ipv4 == 192.168.100.1
```

这些回复把网关 IP 绑定到攻击者 MAC，能够污染受害主机的 ARP 缓存，因而断网现象来自 ARP poisoning。再提取以太网 trailer：

```bash
tshark -r chall.pcap \
  -Y 'arp.opcode == 2 && arp.src.hw_mac == bc:24:11:78:c8:64 && arp.src.proto_ipv4 == 192.168.100.1' \
  -T fields -e eth.trailer
```

相同分片会周期性重发，任取一个完整周期即可。每段首字节是从 `01` 到 `05` 的序号，后面的 8 字节才是 ASCII 数据：

```text
01  Z3JleXtk
02  MWRfMV9q
03  dXM3X2dl
04  N19wMDFz
05  b24zZH0=
```

按序号去重、去掉首字节并连接，得到：

```text
Z3JleXtkMWRfMV9qdXM3X2dlN19wMDFzb24zZH0=
```

进行 Base64 解码：

```python
import base64

data = "Z3JleXtkMWRfMV9qdXM3X2dlN19wMDFzb24zZH0="
print(base64.b64decode(data).decode())
```

输出为：

```text
grey{d1d_1_jus7_ge7_p01son3d}
```

## 方法总结

排查异常流量时应同时回答“为什么断网”和“信息藏在哪里”：异常 ARP Reply 证明了缓存投毒，帧尾 trailer 则承载了隐藏数据。由于攻击帧会重传，不能把所有尾随字节直接拼接；需要利用首字节序号先分组去重，再恢复 Base64 顺序。
