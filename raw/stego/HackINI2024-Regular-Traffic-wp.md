# regular traffic

## 题目简述

附件是一份攻击后的网络抓包。普通流量中夹杂了对 `hacker.com` 的异常 DNS 查询：每个子域都是固定长度十六进制串，同一查询还会因重传重复出现。攻击者把压缩归档切成字节块藏进 DNS 查询名，需要按时间顺序去重、重组并解包。

## 解题过程

### 定位 DNS 隐蔽信道

在 Wireshark 中使用：

```text
dns.qry.name contains "hacker.com"
```

可看到类似查询：

```text
1f8b0800000000000003edce310ac240.hacker.com
1085e1ad3dc5e60232936c920bd8d8d8.hacker.com
...
```

首字节 `1f 8b` 是 gzip magic。查询按抓包时间排列，但相同 label 会重复，所以应保留每个 label 的首次出现，而不是把所有请求盲目连接。

### 重组并解开 gzip 与 tar

下面脚本直接从 PCAPNG 中提取 DNS label，在内存中恢复归档：

```python
from io import BytesIO
import gzip
import tarfile
from scapy.all import DNSQR, rdpcap

packets = rdpcap("capture.pcapng")
seen = set()
chunks = []

for packet in packets:
    if not packet.haslayer(DNSQR):
        continue
    query = bytes(packet[DNSQR].qname).rstrip(b".").decode("ascii")
    if not query.endswith(".hacker.com"):
        continue
    label = query.split(".", 1)[0]
    if label not in seen:
        seen.add(label)
        chunks.append(label)

gzip_data = bytes.fromhex("".join(chunks))
tar_data = gzip.decompress(gzip_data)

with tarfile.open(fileobj=BytesIO(tar_data), mode="r:") as archive:
    member = archive.getmember("flag.txt")
    print(archive.extractfile(member).read().decode())
```

实际附件恢复的是只含 `flag.txt` 的 tar 归档，而不是官方文字说明所称的图片。文件内容为：

```text
shellmates{DNS_3xf1lTr4tI0n!!}
```

这里采用 PCAP 可复现结果；仓库 solution README 中的 flag 拼写与实际载荷不一致。

## 方法总结

- 核心技巧：识别 DNS label 中的十六进制隐蔽信道，按时间顺序去重重组，再依据 magic 逐层解包。
- 识别信号：高熵固定长度子域、同一非常规域名的连续查询、首块出现文件格式 magic，通常意味着 DNS 数据外带。
- 复用要点：DNS 重传会制造重复块；恢复时必须保留顺序并明确去重策略，同时用 gzip 和 tar 解析结果验证边界。
