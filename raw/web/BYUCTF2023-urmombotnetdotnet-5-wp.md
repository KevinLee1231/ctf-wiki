# BYUCTF 2023 - urmombotnetdotnet.com 5

## 题目简述

添加 bot 的接口先用 Python `ipaddress.ip_address` 验证 IP，再写入数据库。`Bots.IP_Address` 实际只有 `VARCHAR(15)`，却没有应用层长度检查。

## 解题过程

Python 的 IPv6 地址允许 `%scope_id`。scope 是接口区域标识，在该版本实现中可以是很长的任意非空字符串，因此下列值仍通过 `ipaddress`：

```text
2001:db8::1000%111111111111111111111111111111111111111111
```

以已登录用户提交：

```json
{"os":"Linux","ip_address":"2001:db8::1000%111111111111111111111111111111111111111111"}
```

写入 15 字符列时触发严格模式异常，调试页面显示 `account_routes.py` 中相邻注释：

```text
byuctf{IPv6_scopes_are_just_arbitrary_strings...maybe_there_are_more_vulns_worldwide?}
```

官方 README 称字段有 256 字符，与仓库 `initial.sql` 的 `VARCHAR(15)` 不符；利用只需超过 15 字符。

## 方法总结

语法有效不代表适合持久化。IPv6 本身已超过 15 字符，再加 scope 更长；数据库字段、应用验证和规范化必须共同支持 IPv6，或明确拒绝 scope 后再按二进制地址存储。
