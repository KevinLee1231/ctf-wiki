# Hackergame2020 从零开始的 HTTP 链接 WP

## 题目简述

题目把 HTTP 服务放在 TCP 0 号端口。浏览器和许多 URL 库把端口 0 当作无效值，直接打开 `http://目标:0/` 会在发包前失败；但 TCP 首部的源、目的端口字段都是 16 位无符号整数，线上报文本身可以表示 0。

这里没有 Web 应用漏洞，关键是绕过客户端对保留端口的策略限制并建立普通 TCP 连接，因此暂归 `_unclassified`。

## 解题过程

在能够连接 0 号端口的 Linux 环境中，用 `socat` 建立本地 TCP 转发：

```sh
socat TCP-LISTEN:20000,fork,reuseaddr TCP:target.example:0
```

该命令在本地监听 20000 端口。浏览器只需要连接正常端口 `http://127.0.0.1:20000/`，`socat` 再把字节流转发到远端 0 号端口。原题 URL 如果带有登录路径或 `token` 查询参数，需要把同一段路径和查询串追加到本地地址，HTTP 层内容不会因 TCP 转发而改变。

服务端返回的 flag 形如：

```text
flag{TCP_P0RT_0_1s_re5erved_BUT_w0rks_<用户相关摘要>}
```

0 号端口虽被 IANA 保留，Linux socket 接口和 TCP 协议仍可构造相应连接。若转发命令本身正常但始终超时，应检查网络路径：部分虚拟机 NAT、家用路由器或防火墙会丢弃目的端口为 0 的流量。把虚拟机改为桥接网络，或从另一台 Linux 主机、云主机发起连接即可。

服务端实际部署时也可以在普通端口运行 HTTP，再用 NAT 规则交换端口，例如：

```sh
iptables -t nat -I PREROUTING -p tcp --dport 20000 -j REDIRECT --to-port 0
iptables -t nat -I PREROUTING -p tcp --dport 0 -j REDIRECT --to-port 20000
```

## 方法总结

“保留”不等于“协议字段无法表示”。遇到特殊端口时，应分清限制来自 URL 解析器、应用程序、操作系统 socket API，还是中间网络设备。本题只需找一个允许目的端口为 0 的底层客户端，再把连接映射到浏览器可接受的本地端口；`socat` 正好完成了这层适配。
