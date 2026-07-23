# choppediver oa

## 题目简述

客户端需在 25 轮中识别走势图为 BUY、SELL 或 HOLD，后期时限缩短到 5 ms，人工不可能完成。WebSocket 服务端在收到 heartbeat 后同步发送较大的 `hb_ack`，而超时检查只发生在 `recv(timeout=...)` 抛出 `TimeoutError` 时。

利用 TCP 流控让服务端阻塞在发送路径，即可绕过应用层截止时间检查。

## 解题过程

每轮开始前先向连接排队发送约 150 个 heartbeat。收到本轮元数据和 PNG 后，将本地 TCP 接收窗口及接收缓冲缩小，例如：

```python
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_WINDOW_CLAMP, 1024)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, 4096)
```

客户端暂时不读取 `hb_ack`，服务端的发送缓冲很快填满，主线程阻塞在同步 `sendall`。此时可以不限时查看走势图并决定答案。

选好答案后，把 answer 以及下一轮要用的 heartbeat volley 都写入客户端发送队列，再恢复较大的接收窗口并持续读取、丢弃积压的 `hb_ack`。服务端解除阻塞后会处理已排队消息，并在再次发生 `recv` 超时前读到 answer。

严格说，单调时钟的 deadline 并没有停止；漏洞是服务端只在 `TimeoutError` 分支比较 deadline，收到实际消息时不重新检查当前时间。因此一个早已过期但已排队的答案仍会被判定。

重复 25 轮后得到：

```text
UMDCTF{cmsc_417_TcP_3nj0y3r}
```

## 方法总结

网络背压会进入应用程序的控制流。同步发送与接收超时放在同一线程时，慢读客户端可以让线程永远到不了计时分支。修复方式包括异步发送、限制 heartbeat 队列、独立计时，以及在处理 answer 时无条件比较当前时间与 deadline。
