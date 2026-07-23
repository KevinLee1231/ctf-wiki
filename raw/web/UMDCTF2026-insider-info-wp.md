# insider-info

## 题目简述

服务端生成 819 个随机英文字母，并把它们每 63 字节分为一个 DNS label，组成隐藏域名。解析器允许查询 `i.inside.info` 的 TXT 记录来获得第 $i$ 个字符；只有查询完整隐藏域名时才返回 flag。

TCP 隧道总共只转发两份 DNS 报文。关键是利用一份 DNS 报文可包含多个 question，而不是把“一次查询”误解成“一个问题”。

## 解题过程

第一份报文把 `QDCOUNT` 设为 819，并依次加入：

```text
0.inside.info TXT
1.inside.info TXT
...
818.inside.info TXT
```

服务端的 `resolve` 会遍历 `request.questions`，并为每个问题追加一条 TXT answer。因此一次隧道请求就能取回全部 819 个字符：

```python
record = DNSRecord(
    header=DNSHeader(id=1),
    questions=[
        DNSQuestion(f"{i}.inside.info", QTYPE.TXT)
        for i in range(819)
    ],
)
```

按 answer 顺序拼接单字节 TXT 数据，再每 63 字节切分为 label。第二份报文需要手工编码 QNAME：

```python
qname = b""
for label in labels:
    qname += bytes([len(label)]) + label
qname += b"\x06inside\x04info\x00"
```

构造一个 `QDCOUNT=1`、类型为 TXT 的标准问题并发送。隐藏域名完全匹配后，解析器返回 `flag.inside.info` 的 TXT 记录：

```text
UMDCTF{5Ur31Y_N0_0N3_W111_N071C3_MY_1N51D3r_7r4D1N6}
```

## 方法总结

限制的是隧道往返次数，不是 DNS question 数量。DNS 头部的 `QDCOUNT` 允许在同一报文中批量查询，先一次性恢复域名素材，再用第二次请求命中完整域名即可。该题的决定性机制是应用层 DNS 报文结构，因此归入 Web/应用协议方向。
