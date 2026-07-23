# A TXT For You and Me

## 题目简述

题目要求查询 `a-txt-for-you-and-me.chall.lol` 的 DNS TXT 记录。TXT 是 DNS 中用于承载任意文本的记录类型，常见用途包括域名验证、SPF/DKIM 策略和题目中的短消息。

这里不需要访问网站，也不应把“域名能否解析出 A 记录”当作成功条件；目标信息直接位于 TXT RRset。

## 解题过程

使用 `dig` 指定记录类型：

```bash
dig +short TXT a-txt-for-you-and-me.chall.lol
```

也可以用 `nslookup`：

```text
nslookup -type=TXT a-txt-for-you-and-me.chall.lol
```

比赛期间，权威 DNS 返回的 TXT 文本为：

```text
UMDCTF{just_old_school_texting}
```

公开仓库中的部署说明进一步确认：该记录原本创建在 GCP 的 `chall.lol` DNS zone 中，题目的全部操作就是对上述完整域名进行 TXT 查询。即使历史 DNS 记录已经下线，WP 仍保留了记录类型、完整查询名和当时的权威结果。

## 方法总结

- 核心技巧：根据题目明确指向 DNS 文本记录，直接查询 TXT，而不是尝试打开 HTTP 服务。
- 识别信号：题名强调 TXT、题面给出完整域名或出现域名验证语义时，应枚举相应 RR type。
- 复用要点：使用 `+short` 便于提取值；若结果为空，再检查末尾点、递归解析器缓存以及记录是否已过期。
