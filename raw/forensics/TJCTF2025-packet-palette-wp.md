# packet-palette

## 题目简述

PCAPNG 中包含一条伪装成 USB-over-IP 的 TCP 会话。控制消息为 `USBIP:DEVICE_READY` 与 `USBIP:DONE`；数据包负载使用 12 字节大端自定义头 `>4sHHI`，字段依次是魔数 `USBI`、分片序号、总分片数和当前分片长度。按序拼接各分片即可恢复一张 PNG。

## 解题过程

与其盲目搜索整个 PCAP 中的 PNG 头，更稳妥的做法是按协议头解析并校验分片。下面的脚本忽略控制包，依据序号重组数据，同时验证长度和分片完整性：

```python
import struct
from scapy.all import Raw, rdpcap

chunks = {}
expected_total = None

for packet in rdpcap("chall.pcapng"):
    if Raw not in packet:
        continue
    payload = bytes(packet[Raw].load)
    if len(payload) < 12 or not payload.startswith(b"USBI"):
        continue

    magic, index, total, length = struct.unpack(">4sHHI", payload[:12])
    chunk = payload[12:12 + length]
    if magic != b"USBI" or len(chunk) != length:
        raise ValueError("invalid fragment")
    if expected_total is not None and total != expected_total:
        raise ValueError("inconsistent fragment count")
    expected_total = total
    chunks[index] = chunk

if expected_total is None or set(chunks) != set(range(expected_total)):
    raise RuntimeError("missing fragments")

image = b"".join(chunks[index] for index in range(expected_total))
assert image.startswith(b"\x89PNG\r\n\x1a\n")
assert image.endswith(b"IEND\xaeB`\x82")

with open("recovered.png", "wb") as f:
    f.write(image)
```

恢复出的图片只显示一行文字，因此直接转写为文本，不保留纯文本截图：

```text
tjctf{usb1p_f13g_1ns1d3_3_pr0t0c0l}
```

## 方法总结

- 核心技巧：识别并解析 PCAP 中的自定义分片协议，按显式序号重组文件。
- 识别信号：重复的固定头、递增索引、总片数和长度字段，以及重组后出现标准文件魔数。
- 复用要点：不能默认抓包顺序就是逻辑顺序；应校验分片长度、重复、缺失和总数，再验证重建文件的头尾标记。
