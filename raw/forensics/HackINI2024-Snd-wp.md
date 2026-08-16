# snd

## 题目简述

附件是从 PCAP 导出的 DNS 流量日志。主机不断查询 `dnsecret.shellmates.club`，其间夹杂 `xtt`、`ttt`、`xxx` 等无效名称；题面又提示“不知道是否拼对”。这些三字符组合是在提示把 `txt` 拼正确，并查询该域名的 DNS TXT 记录。

## 解题过程

日志中的典型模式是：

```text
Standard query ... A xtt
Standard query ... A dnsecret.shellmates.club
Standard query ... A ttt
Standard query ... A dnsecret.shellmates.club
```

`xtt`、`ttx`、`xtx` 等都由 `t`、`x` 组成，但不是有效目标。结合“secret”域名可判断题目希望查询用于承载文本的 TXT record，而不是继续请求 A record。

```bash
dig +short TXT dnsecret.shellmates.club
```

也可以使用：

```bash
nslookup -type=TXT dnsecret.shellmates.club
```

比赛时该 TXT 记录返回：

```text
shellmates{c2_s3RVER_mIgHt_UsE_TXt_R3c0RdS!!}
```

这里的关键证据已经完整写入正文，不依赖该历史 DNS 记录未来仍在线。

## 方法总结

- 核心技巧：从日志中的拼写变体和重复域名识别目标 DNS record type，再主动查询 TXT 记录。
- 识别信号：域名含 `secret`、大量失败的 `xtt/ttx` 请求、题面强调拼写，三者共同指向 `TXT`。
- 复用要点：DNS TXT 常用于验证信息、配置甚至 C2 数据；历史题解应保存返回内容，不能只留下会失效的查询命令。
