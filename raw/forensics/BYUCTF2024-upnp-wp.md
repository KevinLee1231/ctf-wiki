# UPnP

## 题目简述

题目给出路由器 UPnP 的 action 列表以及 `GetDeviceInfo` 返回的二进制消息，要求提取 Base64 编码的 nonce。action 名称指向 `WFAWLANConfig` 服务，消息体实际是 Wi-Fi Protected Setup（WPS）的 TLV 数据。

## 解题过程

WPS 属性由 2 字节大端类型、2 字节大端长度和对应值组成：

```python
from struct import unpack

data = open("msg.bin", "rb").read()
off = 0
while off + 4 <= len(data):
    typ, size = unpack(">HH", data[off:off + 4])
    off += 4
    value = data[off:off + size]
    off += size
    print(hex(typ), size, value)
```

WPS 中 Enrollee Nonce 的属性类型是 `0x101a`。附件中的字段长度为 16，值是原始 nonce 字节：

```text
80 80 c6 5d f6 1a 65 ed 9a 21 ea a1 2c 6c dd 77
```

题目要求 Base64 表示，因此还要编码一次：

```python
import base64
nonce = bytes.fromhex("8080c65df61a65ed9a21eaa12c6cdd77")
print(base64.b64encode(nonce).decode())
```

输出并提交：

```text
byuctf{gIDGXfYaZe2aIeqhLGzddw==}
```

## 方法总结

未知二进制协议先从服务名和 action 识别标准，再严格按字节序解析 TLV。需要区分“字段值是原始 nonce”与“题目要求的 Base64 表示”；本题是读取 16 个原始字节后自行编码。
