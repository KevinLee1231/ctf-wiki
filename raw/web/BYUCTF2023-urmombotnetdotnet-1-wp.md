# BYUCTF 2023 - urmombotnetdotnet.com 1

## 题目简述

这是一个仅有 JSON API 的 Flask/MySQL 应用。注册路由校验用户名是否重复，却没有在写库前检查邮箱唯一性；数据库表对 `Email` 设置了 `UNIQUE`，应用又在调试模式下暴露异常页。

## 解题过程

先用合法数据注册一次，再换一个用户名、复用同一邮箱提交第二次：

```http
POST /api/register HTTP/1.1
Content-Type: application/json

{"email":"same@example.com","username":"user_two","password":"long-password-123","bitcoin_wallet":"0x1234"}
```

第二次 `INSERT` 触发唯一约束异常。Werkzeug 调试 traceback 会显示 `login_routes.py` 的源代码上下文，异常语句上方正有注释：

```text
byuctf{did_you_stumble_upon_this_flag_by_accident_through_a_dup_email?}
```

数据库定义还显示 `Email VARCHAR(128)`、`Bitcoin_Wallet VARCHAR(256)`；官方 README 泛称 255 字符并不精确，重复邮箱是更稳定的触发方式。

## 方法总结

数据库约束不能代替应用层错误处理。即使唯一约束正确阻止重复记录，未捕获异常和调试页面仍会把源码秘密泄漏给客户端；生产环境必须关闭调试器并统一处理完整性错误。
