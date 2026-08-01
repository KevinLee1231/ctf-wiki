# BYUCTF 2022 - Grafana

## 题目简述

Grafana 允许匿名访问一个损坏的体育数据面板。浏览器会把面板的原始 PostgreSQL 查询发送到 datasource API；后端数据库账号又拥有所有表的 `SELECT` 权限。

## 解题过程

打开开发者工具 Network，刷新面板并找到对 `/api/ds/query` 的 POST。请求 JSON 的 `queries[0].rawSql` 原本类似：

```sql
select affiliation_type, count(affiliation_type)
from affiliations
group by affiliation_type;
```

保留请求中的 datasource UID、时间范围和 `format` 字段，只把 SQL 改为：

```sql
SELECT * FROM flag;
```

重新发送请求即可在 JSON 表格响应中读到：

```text
byuctf{qu3ry_1nj3ct10n_1s_4_"f34tur3"_1n_gr4f4n4}
```

仓库 `postgresql_build/init.sql` 说明 `grafana_user` 被授予当前 schema 所有表的读取权限，`sportsdb.sql` 也确实创建并填充了 `flag` 表。这里不是引号逃逸型 SQL 注入，而是匿名用户可控制合法的原始查询功能，且 datasource 权限范围过大。

## 方法总结

分析仪表盘时要检查浏览器实际发送的 datasource 请求。即使前端图表渲染失败，查询仍可能执行；安全边界必须落在后端身份、查询白名单和数据库最小权限上，不能依赖面板 UI 隐藏表名。
