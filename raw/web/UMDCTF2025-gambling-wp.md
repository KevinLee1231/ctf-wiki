# UMDCTF 2025 - gambling

## 题目简述

服务的 HTTP/1.1 端口只返回 `Alt-Svc: h3=...`，真正的注册、兑换、下注和 flag
接口运行在 HTTP/3/QUIC 上。新用户余额为 0；同一个兑换码每次增加 100，购买 flag
需要 300。兑换限速记录“当前请求 IP”，用户认证又要求请求 IP 等于注册 IP，看似
无法从三个地址完成兑换。

漏洞来自 QUIC connection migration：同一逻辑连接可以在请求处理中更换 UDP
源地址，而服务在认证和限速两个时刻分别读取了 `quic_conn.remote_address()`。

## 解题过程

注册时，服务把连接当前地址保存为 `signup_ip`。每个受保护请求先执行：

```rust
if user.signup_ip != quic_conn.remote_address().ip() {
    return FORBIDDEN;
}
```

`/redeem` 的后续顺序却是：

```rust
let user = check_valid_user(...).await?;
let payload = read_payload::<String>(stream).await?;

if user.test_ratelimited(quic_conn.remote_address().ip()) {
    return TOO_MANY_REQUESTS;
}

user.credits.fetch_add(100, Ordering::SeqCst);
```

`read_payload` 会持续等待 DATA，直到客户端结束请求流。因此可以从注册 IP 发出请求头
并发送兑换码数据，让认证先通过；在发送流结束标志前迁移 QUIC endpoint，等服务器
看到新的远端地址后再 `finish()`。限速器此时记录的是迁移后的 IP。

兑换码不需要 Base64 解码，必须作为 JSON 字符串原样提交：

```text
eW91IHRoaW5rIHlvdSdyZSBzcGVjaWFsIGJlY2F1c2UgeW91IGtub3cgaG93IHRvIGRlY29kZSBiYXNlNjQ/
```

官方 Rust 客户端的流程为：

1. 在地址 A 建立 HTTP/3 连接并注册随机用户名、密码；
2. 第一次兑换不迁移，限速集合加入 A，余额变为 100；
3. 重新从 A 开始第二次请求，发出 body 但暂不结束流；将 QUIC endpoint rebind 到
   地址 B，再结束流，余额变为 200；
4. 把连接迁回 A，使下一次认证仍满足 `signup_ip`；第三次请求处理中再迁到全新的
   地址 C，余额变为 300；
5. 迁回 A，以同一 JSON Authorization 头请求 `POST /flag`。

本地模式可分别绑定 `127.0.0.1`、`127.0.0.2` 和 `127.0.0.3`；远程官方脚本在
第二、三次流结束前提示切换到不同 VPN 节点，再切回注册节点。核心不是建立三条新
连接，而是保留同一 QUIC connection ID 并改变其网络路径。余额达到 300 后返回：

```text
UMDCTF{d_D_d_dehH_@W_dANG_!7}
```

用户数据库每 300 秒清空一次，因此操作还需在该窗口内完成。

## 方法总结

本题是连接级身份发生变化造成的 TOCTOU。认证读取一次远端 IP，业务限速在异步读取
body 后又读取一次，却默认两者始终相同；QUIC migration 打破了这个假设。安全实现
应在请求开始时捕获并固定经过认证的身份上下文，限速绑定用户或稳定的会话标识，而
不是在不同 `await` 之后反复把可变的传输层地址当身份。官方浏览器脚本也明确标注为
理论版本；可控 `endpoint.rebind` 的 Rust 客户端才是可直接复现的实现。
