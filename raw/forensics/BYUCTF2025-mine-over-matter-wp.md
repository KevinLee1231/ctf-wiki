# Mine Over Matter

## 题目简述

题目使用与上一题相同的 [BYUCTF 2025 网络日志](https://byu.box.com/s/2rong02xtfx7sfo52nos3ra2waifogv2)，要求找出两台向加密货币矿池通信的内网主机。日志第 19 列是源 IP，第 20 列是目的 IP；官方思路是对所有目的地址做 PTR 反向解析，再把矿池目的地址映射回源主机。

## 解题过程

先收集唯一目的 IP，对每个地址执行 `nslookup` 或 PTR 查询。官方脚本把反向域名中含 `mine` 的记录视为矿池端点，并保存“目的 IP 到 PTR 域名”的映射。再重新扫描日志，凡目的 IP 命中该集合，就记录同一行的源 IP：

```python
import csv
import socket

rows = list(csv.reader(open("logs.txt", newline="")))
destinations = {row[19] for row in rows if len(row) >= 22}

mining_destinations = set()
for ip in destinations:
    try:
        ptr = socket.gethostbyaddr(ip)[0].lower()
    except (socket.herror, socket.gaierror):
        continue
    if "mine" in ptr:
        mining_destinations.add(ip)

miners = {row[18] for row in rows
          if len(row) >= 22 and row[19] in mining_destinations}
print(sorted(miners))
```

结果是 `172.16.0.5` 与 `172.16.0.10`：

```text
byuctf{172.16.0.5,172.16.0.10}
```

题目接受两个地址的任意顺序。PTR 记录可能随时间变化，复现时应把查询结果连同时间一起缓存；本题官方脚本和比赛快照共同确认了上述两台源主机。

## 方法总结

- 核心技巧：用目的 IP 的反向 DNS 名识别矿池基础设施，再沿同一流量记录回溯内网源地址。
- 识别信号：网络日志只给 IP、题目问外部服务类型时，PTR/WHOIS/ASN 可作为富化手段，但最终主机归因仍要回到原始五元组方向。
- 复用要点：变量名不能代替字段定义；官方脚本中曾把目的字段临时命名为 `srcIP`，实际应按第 19/20 列的方向解释。
