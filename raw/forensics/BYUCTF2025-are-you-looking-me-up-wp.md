# Are You Looking Me Up?

## 题目简述

题目给出一份逗号分隔的原始网络日志，要求找出收到 DNS 请求最多的服务器。原始证据由主办方放在 [BYUCTF 2025 网络日志](https://byu.box.com/s/2rong02xtfx7sfo52nos3ra2waifogv2)。日志中第 20 列是目的 IP，第 22 列是目的端口；协议文本可用于筛出 UDP。

## 解题过程

筛选 UDP 且目的端口为 53 的记录，按目的 IP 分组计数：

```bash
awk -F',' 'tolower($0) ~ /udp/ && $22 == 53 {print $20}' logs.txt \
  | sort | uniq -c | sort -nr | head
```

为避免字符串 `grep udp` 匹配到其他字段，也可以用 Python CSV 解析器按真实协议列筛选；核心统计仍是 `(destination_ip, destination_port=53)`。

排名第一的目的地址为 `172.16.0.1`，因此答案是：

```text
byuctf{172.16.0.1}
```

该结论由目的端口和目的地址共同确定，不能把发起查询最多的源主机误报为 DNS 服务器。

## 方法总结

- 核心技巧：对网络日志按 UDP/53 过滤，并对目的 IP 聚合计数。
- 识别信号：题目问“收到最多请求的服务器”时，统计维度应是目的地址；问客户端或发起者时才统计源地址。
- 复用要点：先核对列定义和方向，再计数；正式处理含引号或转义的 CSV 时应使用 CSV 解析器而非简单切分。
