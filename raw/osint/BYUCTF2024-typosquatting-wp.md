# TypoSquatting

## 题目简述

题目给出疑似 Facebook typosquatting 域名 `facebooks.com`，要求域名注册到期日。该信息来自 WHOIS/RDAP 注册数据，而不是网站页面内容。

## 解题过程

对目标域名执行 WHOIS，并关注 registrar 记录中的 Registry Expiry Date，而不是 DNS TTL、证书有效期或网页版权年份：

```text
whois facebooks.com
```

BYUCTF 2024 比赛时的注册记录快照显示：

```text
Registry Expiry Date: 2025-04-10
```

因此提交：

```text
byuctf{2025-04-10}
```

域名可以续费、转移或变更注册商；当前 WHOIS 很可能已有新的到期日。这里记录的是题目发布时的历史证据，不能用 2026 年查询值重写原答案。

## 方法总结

域名 OSINT 要区分创建日、更新日与注册到期日，并优先使用注册局/注册商数据。对时效性字段应保存查询时间和原始字段名，否则后续复现者会被正常的续费变化误导。
