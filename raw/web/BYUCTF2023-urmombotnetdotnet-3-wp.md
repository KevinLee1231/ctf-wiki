# BYUCTF 2023 - urmombotnetdotnet.com 3

## 题目简述

向已有工单追加消息时，应用把旧内容、换行和新消息拼接后写回 `Messages VARCHAR(2048)`，却从不检查累计长度。

## 解题过程

创建一个短工单并记下 `ticket_id`，然后向下列接口重复追加消息：

```http
POST /api/tickets/<ticket_id> HTTP/1.1
Content-Type: application/json
Cookie: token=<jwt>

{"message":"A...A"}
```

源码执行：

```python
new_message = old_messages + "\n" + message
```

因此即使单条恰为 2048 字符，前置换行也会使新值超长；也可以用多条短消息累积触发。数据库异常页泄漏相邻注释：

```text
byuctf{let's_not_even_talk_about_the_newline_injection...}
```

## 方法总结

只校验单次输入长度不足以保护累计字段。应在拼接后的最终值上检查上限，并考虑分隔符本身占用空间；更合理的 schema 是把每条消息拆成独立记录，而不是反复改写一个 VARCHAR。
