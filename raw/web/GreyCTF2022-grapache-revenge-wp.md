# GreyCTF2022 - Grapache Revenge

## 题目简述

Revenge 保留 Apache SSRF 与 Grafana 路径穿越，但 flag 不再直接写入配置。需要读取 Grafana 配置和 SQLite 数据库，解密 ClickHouse 数据源密码，复用凭据登录 Grafana，再利用 ClickHouse 的 URL 表函数访问只监听本机的 flag 页面。

## 解题过程

先用与前题相同的代理 URI 分别读取：

```text
/etc/grafana/grafana.ini
/var/lib/grafana/grafana.db
```

配置给出 `secret_key = SW2YcwTIb9zpOOhoPsMoiID9fd`。SQLite 的 `data_source` 表中记录用户 `grey_user`、数据库 `grey_database` 和加密密码 `MGtT...MtA==`。官方 `AESDecrypt.go` 先取 8 字节 salt，用 PBKDF2-SHA256 迭代 10000 次导出 32 字节密钥，再按载荷标记选择 AES-CFB/GCM；解密得到密码 `jas502n`。

复用该账号登录 Grafana，在 ClickHouse 数据源提交：

```sql
SELECT column1
FROM url('http://localhost', LineAsString, 'column1 String')
```

请求由 ClickHouse 服务发起，能访问仅绑定本机的 HTTP 页面，返回：

```text
grey{wh47_4_l0n6_w4y_1_h4v3_60n3_7hr0u6h_0557d6c45546ef3a}
```

## 方法总结

该链包含两次 SSRF：Apache SSRF 用于接触并穿越 Grafana，ClickHouse `url()` 再访问其本机服务。数据库里的“加密凭据”并非独立安全边界，因为解密密钥也能通过同一文件读取漏洞取得；正文所需算法和参数均已列出，无需依赖外链。
