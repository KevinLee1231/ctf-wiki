# Fujiwara Tofu Shop

## 题目简述

题目会根据客户端提交的 HTTP 请求头逐步给出提示。每满足一项要求，响应中就会出现下一项缺失条件；目标是构造一份同时满足来源页面、车辆标识、口味、油量和客户端 IP 等检查的请求。

## 解题过程

首次访问页面后，按照响应中的提示依次补齐请求头：

- `Referer: qiumingshan.net`：声明请求来自秋名山相关页面。
- `User-Agent: Hachi-Roku`：将客户端标识伪装成 AE86。
- `Cookie: flavor=Raspberry`：提交服务端要求的口味。若响应先通过 `Set-Cookie` 下发该值，也可以让 HTTP 客户端保存后再请求。
- `Gasoline: 100`：补充题目自定义的油量请求头。
- `X-Real-IP: 127.0.0.1`：将反向代理传递给应用的客户端地址设为本机。

可以用一条 `curl` 命令复现完整请求：

```bash
curl 'https://challenge.example/' \
  -H 'Referer: qiumingshan.net' \
  -H 'User-Agent: Hachi-Roku' \
  -H 'Cookie: flavor=Raspberry' \
  -H 'Gasoline: 100' \
  -H 'X-Real-IP: 127.0.0.1'
```

最后一关需要注意题目后端使用 Gin。这里检查的是 Gin 能识别的真实客户端地址，而不是笼统地添加 `X-Forwarded-For`。Gin 在反向代理场景下会结合可信代理配置解析 `X-Forwarded-For` 或 `X-Real-IP`；本题环境接受的是 `X-Real-IP: 127.0.0.1`。这一行为及其安全边界可参见 [Gin 关于客户端 IP 请求头的讨论](https://github.com/gin-gonic/gin/issues/1684)。

满足全部条件后，响应返回：

```text
hgame{I_b0ught_4_S3xy_sw1mSu1t}
```

## 方法总结

这道题考查 HTTP 请求头和逐步反馈式条件绕过。关键不是猜测所有字段，而是每次观察响应中新出现的提示，再精确补齐对应请求头。涉及框架的客户端 IP 判断时，还要区分 `X-Real-IP`、`X-Forwarded-For` 与应用自身的可信代理配置，不能把它们当成完全等价的字段。
