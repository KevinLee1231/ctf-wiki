# Become A Member

## 题目简述

题目要求按照连续提示构造一份“入会申请”。每条提示分别对应常见 HTTP 元数据：身份写入 `User-Agent`，邀请码位于 Cookie，请求来源由 `Referer` 表示，本地访问可由代理转发头表示，账号信息则作为 JSON 请求体发送。所有条件必须同时满足，而且题目特意要求使用带请求体的 `GET` 请求。

## 解题过程

根据页面依次给出的提示，得到以下映射：

| 提示 | 请求内容 |
| --- | --- |
| 身份证明 `Cute-Bunny` | `User-Agent: Cute-Bunny` |
| Vidar 邀请码 | 将初始 Cookie `code=guest` 改为 `code=Vidar` |
| 来自 `bunnybunnybunny.com` | `Referer: http://bunnybunnybunny.com` |
| 本地请求 | `X-Forwarded-For: 127.0.0.1` |
| 账号信息 | JSON：`username=luckytoday`、`password=happy123` |

最终请求可整理为：

```http
GET / HTTP/1.1
Host: 目标地址
User-Agent: Cute-Bunny
Cookie: code=Vidar
Referer: http://bunnybunnybunny.com
X-Forwarded-For: 127.0.0.1
Content-Type: application/json
Content-Length: 52

{"username":"luckytoday","password":"happy123"}
```

这里容易忽略两点：

1. 请求方法必须保持为 `GET`；HTTP 报文允许 `GET` 携带请求体，不能因为要发送 JSON 就擅自改为 `POST`。
2. 请求体要按 JSON 解析，因此必须同时设置 `Content-Type: application/json`。

全部条件满足后，服务端返回：

```text
hgame{H0w_ArE_Y0u_T0day?}
```

## 方法总结

- 核心技巧：把自然语言提示逐项映射为 HTTP 方法、请求头、Cookie 和请求体。
- 关键细节：后端会同时校验多个字段，漏掉任一条件都会失败；`GET` 是否携带 body 与 body 的媒体类型是两个独立问题。
- 复用要点：遇到请求构造题，应先列条件表，再在代理重放器中一次性核对方法、头部、Cookie 和正文，避免只修改可见表单字段。
