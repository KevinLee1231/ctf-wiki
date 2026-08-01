# BYUCTF 2023 - urmombotnetdotnet.com 4

## 题目简述

注册接口要求 Python 字符串长度至少为 4，而登录成功时若提交的用户名长度小于 4 就返回 flag。比赛数据库/驱动对 NUL 字符的存储或比较行为造成了两个校验层之间的不一致。

## 解题过程

注册时用 JSON Unicode 转义发送三个 NUL 加一个 `a`：

```json
{
  "email": "nul@example.com",
  "username": "\u0000\u0000\u0000a",
  "password": "long-password-123",
  "bitcoin_wallet": "0x1234"
}
```

Flask 解析后 `len(username) == 4`，通过注册检查；比赛部署中的 MySQL 客户端/字符串处理最终让记录可用短用户名 `a` 匹配。随后登录：

```json
{"username":"a","password":"long-password-123"}
```

认证成功且本次 Python 输入长度为 1，响应中的 `flag` 字段为：

```text
byuctf{I_used_unicode_to_make_a_username_under_4_chars_wbu?}
```

该行为依赖比赛镜像的数据库与驱动组合，不应泛化为“MySQL 总会忽略 Unicode/NUL”。仓库使用未固定版本的 MySQL 镜像，后续环境可能不再复现。

## 方法总结

身份字段必须在应用、驱动和数据库之间使用一致的规范化和字符约束。长度检查应针对最终存储/比较形式，并拒绝控制字符；否则可产生注册名与登录名不一致的别名账户。
