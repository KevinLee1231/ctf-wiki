# Stalking 3

## 题目简述

WelcomeCTF2021 的 Stalking 3 使用上一关得到的域名 `sitdownnow.tk`。网站已经无法访问，题面提示转向 DNS，目标是从公开 DNS 记录中恢复信息。

## 解题过程

先查询域名的权威名称服务器。比赛时记录显示其权威服务器为 `ns1.digitalocean.com`。随后直接向权威服务器查询 TXT 记录：

```bash
dig @ns1.digitalocean.com sitdownnow.tk TXT
```

TXT 应答中保存了 flag：

```text
greyhats{7h15_1Nf0Rm4T10n_1s_4vA1l4bl3_T0_Th3_PuBlic}
```

这里不需要恢复网页内容。DNS 是独立于 HTTP 服务的公开数据面，即使站点下线，域名的 NS、TXT、MX 等记录仍可能泄露线索。

## 方法总结

网站不可访问时不应立即判定线索失效。应把域名拆成多个可查询层面：注册信息、名称服务器、DNS 记录、证书透明度和历史快照。本题明确指向 DNS，直接查询权威服务器能减少递归解析器缓存或过滤带来的不确定性。
