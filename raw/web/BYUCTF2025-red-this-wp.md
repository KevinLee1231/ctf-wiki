# Red This

## 题目简述

Flask 前端通过内部 Redis 保存用户、密码、语录和管理员选项。`POST /get_quote` 直接把表单字段 `famous_person` 传给 `getData(key)`；非管理员唯一限制是键名不能包含 `flag`。密码键采用 `<username>_password` 命名，而登录逻辑也从 Redis 读取该键。

因此无需攻击 Redis 协议，Web 层已经暴露了任意键读取原语。

## 解题过程

以普通会话提交：

```http
POST /get_quote HTTP/1.1
Content-Type: application/x-www-form-urlencoded

famous_person=admin_password
```

该键名不包含 `flag`，会直接进入 `db.get`。响应页面显示管理员明文密码。使用返回值登录 `/login` 后，session 中的用户名变为 `admin`。

首页调用 `getAdminOptions("admin")`，从 Redis JSON 键 `admin_options` 读取管理员专属选项，其中暴露了实际 flag 键名。再次向 `/get_quote` 提交这个键；管理员身份跳过字符串过滤，得到：

```text
byuctf{al1w4ys_s2n1tize_1nput-5ed1s_eik4oc85nxz}
```

官方 Postman 图片只展示了 `famous_person=admin_password` 及其文本响应，这些信息已经完整转写，不再保留图片。

## 方法总结

- 核心技巧：把未约束的 Redis 键查询先用于读取管理员密码，再借管理员页面取得被过滤的 flag 键。
- 识别信号：服务端根据用户提供的键名直接查询键值数据库，而敏感数据与普通数据共享命名空间时，黑名单通常可被间接信息链绕过。
- 复用要点：使用服务器端白名单映射公开选项，不要让客户端控制真实数据库键；密码也不应以明文存储。
