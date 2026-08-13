# PCAP 3

## 题目简述

PCAP 中包含大量发往 `8.8.8.8` 的 DNS 查询。查询名的左侧标签由短 Base64 片段组成，并统一追加 `.welc0mectf.gr3yh4ts.com`，形成 DNS 数据外传通道。目标是按包序重组这些标签并解码敏感文本。

## 解题过程

先只保留发往目标解析器、且属于指定后缀的 DNS 请求：

```bash
tshark -r exfiltration.pcap \
  -Y 'dns.flags.response == 0 && ip.dst == 8.8.8.8 && dns.qry.name contains "welc0mectf.gr3yh4ts.com"' \
  -T fields -e dns.qry.name
```

抓包中共有 209 个相关查询。逐行去掉后缀，保持原包顺序拼接，再做一次 Base64 解码：

```python
from base64 import b64decode

suffix = ".welc0mectf.gr3yh4ts.com"
with open("queries.txt", encoding="utf-8") as f:
    chunks = [line.strip().removesuffix(suffix) for line in f if line.strip()]

message = b64decode("".join(chunks)).decode()
print(message)
```

重组后得到 1554 个字符的正文，末尾为：

```text
grey{dn5_3xf11724710n_15_c001}
```

## 方法总结

DNS 外传的典型信号是同一长后缀下出现大量高熵、短而连续的子域标签。恢复时要固定请求方向、过滤响应与正常 DNS 噪声、按捕获顺序拼接，并在最后再判断 Base64、hex 等表示层编码；顺序错误会让整体解码失败。
