# PCAP 3

## 题目简述

附件记录了攻击者通过 DNS 查询外带数据的过程。每个查询名携带一段编码文本，并使用固定域名后缀。目标是筛选外带流量、按捕获顺序拼接片段并解码。

## 解题过程

官方解法先筛选发往公共 DNS 服务器的分组：

```text
ip.dst == 8.8.8.8
```

这些 DNS 查询名形如：

```text
<fragment>.welc0mectf.gr3yh4ts.com
```

将过滤结果按捕获顺序导出为 CSV，读取 `Info` 列中的查询名，去掉固定后缀并依次连接，最后进行 Base64 解码：

```python
import base64
import csv

suffix = ".welc0mectf.gr3yh4ts.com"
parts = []

with open("filtered.csv", newline="", encoding="utf-8") as f:
    for row in csv.DictReader(f):
        query = row["Info"].split()[4]
        parts.append(query.removesuffix(suffix))

print(base64.b64decode("".join(parts)).decode())
```

解码出的文本包含：

```text
grey{dn5_3xf11724710n_15_c001}
```

也可以在表格软件中拆分 `Info` 列、批量去除后缀并用 `CONCAT` 连接，但核心仍是保持原始分组顺序。

## 方法总结

DNS 外带常把数据塞进子域标签，而权威域后缀保持不变。分析时要同时关注过滤条件、片段顺序、固定后缀与外层编码；任何一次乱序或漏包都会让最终 Base64 解码失败。
