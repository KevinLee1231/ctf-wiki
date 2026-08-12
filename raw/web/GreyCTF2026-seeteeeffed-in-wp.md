# SeeTeeEffedIn

## 题目简述

Flask 应用为每个新注册玩家创建一条 `secrets(owner_player_id, flag)` 记录，并使用 PostgreSQL RLS：会话处理先执行 `set_config('app.player_id', <当前玩家>, true)`，所以玩家只能读取自己的 flag。用户名更新本身使用参数化 SQL，但 `player_usernames` 上的 PostgreSQL `refint` 级联更新触发器在受影响版本中会把新值拼接进动态 SQL，重新引入 SQL 注入。

目标不是越过 RLS 读取别人的数据，而是通过自己的 RLS 上下文将自己的 `secrets.flag` 写回 API 会返回的 `user_sessions.session_note`。

## 解题过程

### 找到可见的回显字段与触发器

注册调用 `register_player(...)`，数据库函数在玩家自己的 `secrets` 行写入当前挑战 flag，并返回 session token。`/api/me` 及重命名成功响应都会调用 `fetch_session`，其中包含：

```sql
SELECT s.session_token, s.player_id, s.username, s.session_note
FROM user_sessions AS s
...
```

Schema 中 `player_usernames` 的触发器为：

```sql
CREATE TRIGGER player_usernames_refint_cascade
AFTER UPDATE OR DELETE ON player_usernames
FOR EACH ROW
EXECUTE FUNCTION check_foreign_key(
    1, 'cascade', 'username', 'user_sessions', 'username'
);
```

受影响的 `refint` cascade-on-update 实现将新 username 未转义地嵌入等价于 `UPDATE user_sessions SET username='<new value>' ...` 的动态 SQL。应用层对 `/api/profile/private-rename` 的参数化只保护了最外层 `UPDATE`，保护不到 trigger 内部二次拼接。

### 注入到 session_note

先用两个随机且互不冲突的 public/private username 注册，保存 session token。设 public username 为 `pub`，向私有用户名重命名接口提交：

```text
pub' , session_note = (SELECT flag FROM secrets) -- -
```

触发器拼接后，关键部分变为：

```sql
UPDATE user_sessions
SET username='pub' , session_note = (SELECT flag FROM secrets) -- -'
WHERE username = ...
```

把 `username` 同时设回真实的 public handle 有两个作用：它保留了最后的 `check_primary_key` 引用完整性要求，也避免破坏当前 session。子查询运行前，应用已经为该连接设定 `app.player_id`；RLS 将 `SELECT flag FROM secrets` 自动限制在新注册玩家自己的那一行，故无需猜测其他用户 ID。

重命名响应中的 `data.session_note` 就是 flag；也可用同一 `X-Session-Token` 请求 `/api/me` 复核。官方 exploit 的成功值为：

```text
grey{refint_cascade_update_sql_injection}
```

## 方法总结

- 核心技巧：应用层参数化正确，但第三方数据库 trigger 的动态 SQL 未转义，形成级联更新 SQL 注入。
- 识别信号：用户输入进入会触发外键/引用完整性 callback 的列，且服务响应又暴露可写 session/profile 字段；RLS 上下文在同一连接中已建立。
- 复用要点：审计 SQL 注入不能只看 Web handler，还要检查 trigger、extension 与 `SECURITY DEFINER` 函数。RLS 会限制注入的读范围，但在本题中“读取自己被隐藏的字段”已经足够完成目标。
