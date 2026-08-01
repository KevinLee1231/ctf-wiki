# BYUCTF 2023 - TOR

## 题目简述

给定 Tor relay 指纹 `301612857BAB556E60BB035FF1C9C76688BFED4A`，要求恢复曾与它关联的 OR 地址，不需要访问 Tor 网络。

## 解题过程

在 [Tor Metrics Relay Search](https://metrics.torproject.org/rs.html#details/301612857BAB556E60BB035FF1C9C76688BFED4A) 中按完整指纹查询，可确认节点昵称为 `BridgeTest1`，但页面不再给出 OR 地址。

继续用同一不可变指纹查询历史聚合站 [Onionite 节点记录](https://onionite.net/node/301612857BAB556E60BB035FF1C9C76688BFED4A)，记录中的 OR 地址是：

```text
10.54.14.33:55253
```

因此：

```text
byuctf{10.54.14.33:55253}
```

这是 RFC 1918 私网地址，看起来异常，但正是历史记录给出的值；不应因为“不像公网 relay”而擅自改写。

## 方法总结

Tor 指纹是跨数据源关联的稳定主键。当前官方页面缺字段时，可用历史聚合数据补全，但要同时保留节点昵称、指纹和来源，防止仅凭相似 IP/端口误关联。
