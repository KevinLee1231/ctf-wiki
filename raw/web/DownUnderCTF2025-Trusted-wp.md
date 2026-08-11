# Trusted

## 题目简述

本题同样延续 `Horoscopes`。公开 Gemini capsule 的 `community-hub.gmi` 泄露管理端口 `756f`，`survival.gmi` 泄露每日问答口令。管理服务实现了 Gemini 风格的 URL 路由和状态码，却在部署时没有启用 TLS，因此标准 Gemini 客户端握手失败，需要通过原始 TCP 手工构造请求。

## 解题过程

### 恢复端口与口令

`756f` 是十六进制：

$$
0x756f=30063.
$$

公开页面还给出：

```text
Prompt:   Moonlight reflects twice on still water
Response: But+ripples+show=truth%in motion
```

服务端容器直接执行 `./admin`，没有传入启用 TLS 的参数，所以对端口 30063 使用 `nc` 或 `socat`。请求 `/` 可看到 `password_protected.gmi` 路由；请求该路径但不带 query 时，服务器返回 Gemini 输入状态 `11` 和提示语。

### URL 编码提交口令

服务端读取 `u.RawQuery` 后调用 `url.QueryUnescape()`。原口令包含字面量 `+`、`=`、`%` 和空格，必须按 query 规则编码：

```text
But%2Bripples%2Bshow%3Dtruth%25in+motion
```

向原始 TCP 连接发送：

```text
/password_protected.gmi?But%2Bripples%2Bshow%3Dtruth%25in+motion\r\n
```

可用如下命令复现请求：

```bash
printf '/password_protected.gmi?But%%2Bripples%%2Bshow%%3Dtruth%%25in+motion\r\n' |
  nc <challenge-host> 30063
```

响应状态变为 `20 text/gemini`，正文返回：

```text
DUCTF{Cr1pPl3_Th3_1nFr4sTrUCtu53}
```

## 方法总结

- 核心技巧：从公开 capsule 关联端口和口令，识别部署未启用 TLS，再手工实现最小 Gemini 请求并正确编码 query。
- 识别信号：标准客户端报 TLS 失败，但裸 TCP 能读到 `20`、`11`、`59` 等 Gemini 状态；源码或页面还泄露十六进制端口与每日口令。
- 复用要点：协议实现和部署方式必须分开核对。提交 query 时区分字面量加号与代表空格的 `+`，并编码 `%`、`=` 等保留字符，否则 `QueryUnescape` 会报错或得到不同字符串。
