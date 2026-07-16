# Http的真理，我已解明

## 题目简述

服务端依次检查 URL 查询参数、POST 表单、Cookie 和三个 HTTP 请求头。`User-Agent: Safari` 与 `Via: clash` 只是要求请求头字符串等于指定值，并不要求真实使用 Safari 浏览器或 Clash 代理；`Referer` 同理可以由客户端直接设置。

## 解题过程

源码要求的有效请求包含：

- GET 参数 `hello=web`；
- POST 表单字段 `http=good`；
- Cookie `Sean=god`；
- 请求头 `User-Agent: Safari`；
- 请求头 `Referer: www.mihoyo.com`；
- 请求头 `Via: clash`。

前三项的源码把“未设置”和“值不匹配”用 `&&` 连接，因此实际只要参数存在就不会进入拒绝分支；使用题目提示的正确值最直观。三个请求头则是直接比较，必须精确匹配。

使用命令行可以直接构造最小请求，不需要复制浏览器生成的大量无关头部：

```bash
curl -X POST 'http://<host>/?hello=web' \
  --data 'http=good' \
  --cookie 'Sean=god' \
  -H 'User-Agent: Safari' \
  -H 'Referer: www.mihoyo.com' \
  -H 'Via: clash'
```

源码中的 `X-Forwarded-For` 检查已经被注释，因此不应再围绕 XFF 构造请求。满足现有六项条件后，服务端直接输出 flag。

## 方法总结

- 核心技巧：理解 HTTP 请求各组成部分，并按服务端实际读取方式精确构造。
- 识别信号：`$_GET`、`$_POST`、`$_COOKIE` 与 `$_SERVER['HTTP_*']` 连续比较固定值。
- 复用要点：请求头名称只是协议字段，通常可以手工设置；以当前源码为准，删除或注释掉的检查不应继续浪费时间。
