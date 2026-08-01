# proxyport

## 题目简述

题目要求连续判断公开端口背后使用的是 GOST 还是 FRP。两个候选都会把连接转发到同一个 Caddy HTTP 服务，正常请求得到的内容完全相同，因此无法依靠响应正文、状态码或应用层握手区分。

真正的差异位于 TCP 连接关闭语义。正常的半关闭只终止一个方向：客户端发送 FIN 后，客户端到服务端的数据流结束，但服务端仍应能够沿反方向发送剩余响应。FRP 的 TCP 转发没有完整保持这一状态，收到客户端 FIN 后会把两端都视为关闭，使后端响应无法继续返回；GOST 则仍能把 Caddy 的响应送回客户端。

每轮最多进行 5 次探测。利用这一半关闭差异即可构造稳定指纹，再提交 `answer frp` 或 `answer gost`。

## 解题过程

### 1. 排除应用层指纹

无论后端是 GOST 还是 FRP，发送普通 HTTP 请求都会得到相同的 Caddy 响应：

```http
HTTP/1.1 200 OK
Server: Caddy
Content-Length: 5

caddy
```

这说明公开端口是透明 TCP 转发入口。继续更换 HTTP 方法、请求路径或报文内容只会改变 Caddy 的行为，无法暴露转发器类型。需要主动构造 TCP 状态变化，观察数据方向的关闭是否符合半关闭语义。

### 2. 构造半关闭探测

建立 TLS 连接后，先发送一份完整的 HTTP/1.0 请求：

```http
GET / HTTP/1.0
Host: <target>

```

随后调用 `CloseWrite()` 结束客户端写方向，但继续读取连接：

```go
conn, err := tls.DialWithDialer(
    &net.Dialer{Timeout: 3 * time.Second},
    "tcp",
    addr,
    &tls.Config{},
)
if err != nil {
    return err
}
defer conn.Close()

conn.SetWriteDeadline(time.Now().Add(2 * time.Second))
if _, err := fmt.Fprintf(
    conn,
    "GET / HTTP/1.0\r\nHost: %s\r\n\r\n",
    addr,
); err != nil {
    return err
}

if err := conn.CloseWrite(); err != nil {
    return err
}

conn.SetReadDeadline(time.Now().Add(800 * time.Millisecond))
response, err := io.ReadAll(conn)
```

结果分为两类：

- GOST：客户端写方向关闭后，后端到客户端方向仍可继续传输，能够读到 HTTP 响应。
- FRP：FIN 触发整条转发连接清理，读取很快以 EOF 结束且响应为空。

超时与网络错误不能直接当作 FRP；只有“没有响应字节，并且读取以正常 EOF 结束”才是目标特征。

### 3. 五次并发探测

每轮并发执行 5 次相同探测。任意一次出现空响应并正常结束，就记录 FRP 特征：

```go
func identifyService(addr string) (string, error) {
    var (
        wg         sync.WaitGroup
        confirmFrp atomic.Bool
    )

    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()

            conn, err := tls.DialWithDialer(
                &net.Dialer{Timeout: 3 * time.Second},
                "tcp",
                addr,
                &tls.Config{},
            )
            if err != nil {
                return
            }
            defer conn.Close()

            conn.SetWriteDeadline(time.Now().Add(2 * time.Second))
            if _, err := fmt.Fprintf(
                conn,
                "GET / HTTP/1.0\r\nHost: %s\r\n\r\n",
                addr,
            ); err != nil {
                return
            }

            if err := conn.CloseWrite(); err != nil {
                return
            }

            conn.SetReadDeadline(time.Now().Add(800 * time.Millisecond))
            response, err := io.ReadAll(conn)
            if err != nil {
                return
            }
            if len(response) == 0 {
                confirmFrp.Store(true)
            }
        }()
    }

    wg.Wait()
    if confirmFrp.Load() {
        return "frp", nil
    }
    return "gost", nil
}
```

完成当轮探测后提交对应答案，重复直到所有轮次结束并取得 flag。

## 方法总结

- 核心技巧：通过 TLS `CloseWrite()` 制造 TCP 半关闭，检查后端到客户端方向能否在收到 FIN 后继续传输，以此区分两个应用层响应完全一致的透明转发器。
- 识别信号：当候选服务共享相同后端、正常请求没有内容差异时，应检查 FIN、RST、EOF、双向复制和连接回收等传输层状态，而不是继续枚举 HTTP 特征。
- 复用边界：应用层代理未完整实现 TCP 状态机的问题并不少见，但该现象不能单独证明某个端口一定由 FRP 提供；它只是在题目给定的二选一环境中形成有效指纹，实际网络测绘仍需结合版本、部署方式和其它独立证据。
