# WebSocket VPN

## 题目简述

服务在 `/connect` 把 WebSocket 二进制消息当作原始 IPv4 包注入一个 gVisor TCP/IP 栈，并在每位客户端的私有虚拟网络中监听 `10.0.0.1:80`。该内部 HTTP 服务会直接返回 flag。题目的关键是按协议实现一个客户端，而不是利用内存错误，归入 Web。

## 解题过程

服务连接后先发送 JSON 提示，说明内部主机地址为 `10.0.0.1:80`。之后它要求双向 WebSocket 消息均为二进制，并把客户端消息作为 IPv4 payload 注入网卡；从虚拟网卡读到的 IPv4 包再原样写回 WebSocket。

官方客户端复现了所需最小网络栈：创建支持 IPv4、ARP、TCP 与 ICMP 的 gVisor stack，给本地 NIC 配置 `10.0.0.2/8` 和默认路由；一个 goroutine 将栈发出的包写成 WebSocket binary message，另一个 goroutine 将收到的 binary message 注回栈。最后让 `http.Transport.Dial` 改用 `gonet.DialTCP`，向内部地址发起普通 HTTP GET。

逻辑链为：

```text
WebSocket binary frame <-> gVisor IPv4 stack <-> TCP connection to 10.0.0.1:80 <-> HTTP response
```

内部 HTTP 响应体即为：

```
DUCTF{websockets_rule_Ahquo8Gahnetich0}
```

## 方法总结

“VPN over WebSocket”题不要把二进制帧误当成应用层 JSON 或普通代理数据。先确认服务接收的是哪一层的数据包，再在客户端补齐该层最小栈与地址配置。这里没有扫描、绕过认证或漏洞利用；只要按源码规定把 IPv4/TCP/HTTP 流量正确封装到 WebSocket，内部服务就会正常响应。
