# UMDCTF 2018 - Crypticker

## 题目简述

题目列出多家公司的名称，并提示 Lumpus 同时关注股票和密码学。每个名称都对应一个股票代码，需要把代码按顺序拼接成消息。

## 解题过程

逐项查询或识别股票代码：

```text
United Mobility Technology AG          -> UMD
Nuveen Long/Short Commodity Tot        -> CTF
—                                     -> -
Store Capital Corp                     -> STOR
ING Groep NV                           -> ING
AFLAC Incorporated                     -> AFL
American Graphite Technologies Inc     -> AGIN
Meta Financial Group Inc               -> CASH
```

按原顺序连接：

```text
UMD CTF - STOR ING AFL AGIN CASH
```

恢复分组并套用赛事格式：

```text
UMDCTF-{STORINGAFLAGINCASH}
```

该字符串的 SHA-1 为：

```text
0886e6d8f0e3292e9bda83e5803c0a2bf1f8a144
```

与仓库 `README.md` 中的摘要一致。

## 方法总结

标题中的 `ticker` 是明确线索。遇到公司名称列表时，应优先将公司映射到交易代码，并保留题面中的分隔符；代码连接后的可读性通常会进一步确认分组是否正确。
