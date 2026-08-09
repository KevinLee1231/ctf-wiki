# tcpdump

## 题目简述

抓包中的多段 TCP 载荷共同组成一张 PNG。单个分组没有完整文件，必须按流顺序重组原始 payload。

## 解题过程

锁定承载数据的 TCP 会话，按序列号排序并处理重传，只拼接每段新增的 `Raw` 字节。重组结果应以 PNG 签名开头，并在末尾包含完整 `IEND` 块：

```python
payload = b"".join(ordered_unique_tcp_payloads)
assert payload.startswith(b"\x89PNG\r\n\x1a\n")
assert b"IEND" in payload
open("recovered.png", "wb").write(payload)
```

打开恢复出的图片，转写其中的文字：

```text
n00bz{D1D_Y0U_GET_EVERYTH1NG_!?}
```

## 方法总结

PCAP 文件恢复不能简单按显示顺序复制字段；乱序和重传会导致重复或错位。应以单一 TCP 流、序列号和文件结构三重校验重组结果。
