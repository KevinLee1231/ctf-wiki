# week2Come to Inject me

## 题目简述

登录接口把用户名和密码直接拼接进双引号包裹的 MySQL 查询，预期解法是使用双引号闭合、注释代替空格并构造恒真条件。仓库版本还存在更直接的返回值判断错误：只要 SQL 语法正确就会返回 flag。

## 解题过程

仓库中的查询为：

```php
$sql = "select * from test.0xgame_users "
     . "where username = \"$username\" and password = \"$password\"";
$result = mysqli_query($conn, $sql);
if ($result) {
    die("0xGame{3a5y_w4n_n3ng_m1m4}");
}
```

对于 `SELECT`，`mysqli_query()` 在语句成功执行时会返回 `mysqli_result` 对象，即使结果集为零行也仍为真。源码没有调用 `mysqli_num_rows()`，所以在仓库版本中提交任意不会破坏 SQL 语法的普通值即可得到 flag；这是比注入更直接的逻辑漏洞。

比赛部署与原 WP 描述了额外过滤，并以是否查到用户作为判断条件。按该预期逻辑，可以让用户名闭合双引号并注释后续条件：

```text
username="or/**/1#&password=1
```

`/**/` 在 MySQL 中充当空白，`#` 注释掉后面的密码比较。拼接后的有效部分为：

```sql
select * from test.0xgame_users where username = "" or 1
```

恒真条件会返回用户记录。无论走仓库中的直接逻辑缺陷，还是比赛部署的预期注入链，得到的 flag 均为：

```text
0xGame{3a5y_w4n_n3ng_m1m4}
```

## 方法总结

源码审计时不能只看到字符串拼接就停止，还要继续检查查询结果如何被使用。本题同时暴露了两类问题：未参数化 SQL 导致注入，以及把“查询执行成功”误当成“凭据匹配成功”。修复应使用预处理语句，并明确检查结果行数和账户字段。
