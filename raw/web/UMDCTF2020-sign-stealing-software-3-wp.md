# Houston Astros' Sign Stealing Software - Part 3

## 题目简述

前端只提供固定球队下拉框，但 API 接受客户端提交的任意 `team` 和 `player`，并以等值条件查询 MongoDB。数据库中藏有一个不在下拉框里的异常 `teamID`，其记录的 `playerID` 就是 flag。

## 解题过程

检查 `Batting.json` 或枚举 API 结果，可以找到异常记录：

```json
{
  "playerID": "UMDCTF-{Y0u_N0(SqL)_Ev3ry71nG}",
  "teamID": "9c222530fc5822720b24a18e0c5200957fcbe169589915c963e765c99c7f5f3a",
  "H": "1337",
  "yearID": "1337"
}
```

下拉框只是客户端限制。直接向接口发送：

```http
POST /api/v1/resources/playerdata HTTP/1.1
Content-Type: application/json

{"team":"9c222530fc5822720b24a18e0c5200957fcbe169589915c963e765c99c7f5f3a","player":"all"}
```

返回记录中的 flag 为：

```text
UMDCTF-{Y0u_N0(SqL)_Ev3ry71nG}
```

## 方法总结

前端枚举值不能作为服务端授权边界，用户可以直接改写请求体。本题并不是 NoSQL 注入；真正问题是隐藏对象仍可被未授权的任意标识符查询。
