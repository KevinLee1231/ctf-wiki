# BYUCTF 2023 - urmombotnetdotnet.com 2

## 题目简述

创建工单接口 `/api/tickets` 只检查 `description` 是字符串，没有限制长度；数据库列 `Support_Tickets.Description` 是 `VARCHAR(2048)`。

## 解题过程

先注册并登录，保留响应中的 `token` Cookie。随后提交超过 2048 字符的描述：

```http
POST /api/tickets HTTP/1.1
Content-Type: application/json
Cookie: token=<jwt>

{"description":"A...A"}
```

在比赛的严格 MySQL 配置中，超长值引发 `DataError`，而不是静默截断。调试 traceback 展示 `ticket_routes.py` 的插入语句及其上方注释：

```text
byuctf{oof_remember_to_check_length_limit}
```

## 方法总结

长度约束必须在进入数据库前与 schema 保持一致，并把数据库异常转成普通 4xx 响应。是否报错还受 SQL mode 影响，所以复现时应使用比赛容器配置，而不能假设所有 MySQL 都会同样处理超长字段。
