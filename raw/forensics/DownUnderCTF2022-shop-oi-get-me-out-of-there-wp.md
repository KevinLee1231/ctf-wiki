# DownUnderCTF 2022 Shop-Oi! Get out of there! Writeup

## 题目简述

题目说明攻击者已经进入商店管理员账号，要求从 `DownUnderShop.JSON` 中恢复管理员被修改前的密码。答案不使用 `DUCTF{}` 包裹，且不区分大小写。

日志不会直接记录明文密码，但 `/changepassword` 流程会重定向到带有 Base64 `ref` 参数的登录 URL；同时，邮件日志会记录密码修改通知。需要用时间和来源 IP 把 Web 事件与管理员邮件关联起来。

## 解题过程

前一道日志题已经确定可疑来源是 `58.164.62.91`。筛选该 IP 对 `/changepassword` 的访问以及随后出现的 `/login?ref=`，再与主题为 `Your shop.downunderctf.com Password Has Been Changed` 的邮件按时间关联。

在原始日志时间 `2021-01-01T09:26:52.000+0000`，同一秒出现了：

- `58.164.62.91` 从修改密码页面跳转到带 `ref` 的登录页；
- 发往 `admin@shop.downunderctf.com` 的密码变更通知；
- 同秒还有投递到 `cake@shop.downunderctf.com` 的记录，可视为关联邮件流中的副本。

对 URL 百分号解码，再对 `ref` 执行 Base64 解码，得到两个 MD5 摘要：

```text
3dc919de186d1a8ee62fff92d80839f7:6d7c5b3e796d833b3fdd40f4ce57facd
```

检查其余被该攻击者修改的账号，可以发现冒号右侧始终是同一个值，因此右侧是攻击者统一设置的新密码摘要，左侧才是各账号原密码摘要。对管理员对应的左侧 MD5 做字典恢复，得到：

```text
3dc919de186d1a8ee62fff92d80839f7 -> ozzieozzieozzie
```

可在本地重新计算 MD5 验证候选值，而不必依赖在线查询结果。最终答案为：

```text
ozzieozzieozzie
```

## 方法总结

本题考查跨日志源关联和字段语义判断。先用攻击源 IP 与时间戳把 HTTP 重定向、密码修改动作和邮件通知拼成同一事件，再利用多个账号共享右侧摘要这一规律区分旧密码与新密码。散列恢复结果还应本地重算校验，不能把在线字典服务的返回值当作未经验证的结论。
