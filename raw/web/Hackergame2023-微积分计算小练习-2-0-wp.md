# 微积分计算小练习 2.0

## 题目简述

练习网站提交答案后允许留下不超过 25 个字符的评论。评论过滤了 `&`、`>`、`<`、单引号、圆括号、反引号、点号、逗号和 `%`，却允许双引号；服务端把评论直接格式化进 JavaScript 字符串。Bot 会以与选手相同的用户编号登录，在 `http://web` 域写入 JavaScript 可读、经过 `quote_plus` 编码的 flag cookie，然后访问选手提供的 HTML。

决定性漏洞是 JavaScript 字符串注入配合 `window.name` 的跨页面载荷传递，属于 Web XSS/浏览器 Bot 利用，因此归入 web。

## 解题过程

### 初始化共享状态

`MEMORY` 只按用户编号索引，不区分普通会话与同编号的 `:bot` 会话。`/result` 只有在该编号的 `score` 非空时才展示，因此先用自己的会话向 `/` 提交一次五题答案；是否答对不影响利用，只要创建成绩记录即可。

随后把评论设置为：

```text
"+{a:location=name}+"
```

它没有命中黑名单且不超过 25 字符。服务端原本生成：

```javascript
updateElement("#comment", "你留下的评论：COMMENT");
```

代入后变成合法表达式，其中对象字面量的值会执行 `location=name`。由于 `location` 和 `name` 都是 `window` 的全局属性，这里无需使用被过滤的点号。

### 用 `window.name` 携带长脚本

评论本身太短，真正的脚本放在 Bot 会访问的攻击者 HTML 中。`window.open(url, target)` 会把第二个参数保存为新窗口的 `window.name`，而页面跨源导航后该值仍然存在。Bot 允许弹窗，因此可提交如下 HTML；其中 `START`、`END` 每次替换为要读取的 cookie 区间：

```html
<script>
window.open(
  "http://web/result",
  'javascript:fetch("/result", {"headers":{"Content-Type":"application/x-www-form-urlencoded"},"body":`comment=${btoa(document.cookie.substring(START,END))}`,"method":"POST"});'
);
</script>
```

流程如下：

1. 弹窗先打开同一用户编号的 `http://web/result`，并带着以 `javascript:` 开头的长 `window.name`。
2. 结果页渲染预先保存的短评论，执行 `location=name`。
3. 浏览器把 `window.name` 当作 `javascript:` URL 执行；此时脚本运行在 `http://web` 源，可以读取 flag cookie。
4. 脚本把 cookie 片段 Base64 编码，再 POST 回同源 `/result`，覆盖该用户编号的评论。
5. 选手刷新自己的结果页，即可读回这一段 Base64。

评论最多 25 字符，取 18 个 cookie 字节会编码成 24 个 Base64 字符。每取完一段，Bot 已把原 XSS 评论覆盖成泄露结果，所以在读取下一段前必须重新提交 `"+{a:location=name}+"`，再用新的区间触发 Bot。

`application/x-www-form-urlencoded` 会把 Base64 中的 `+` 解析成空格；读回时应先把空格还原为 `+`。逐段 Base64 解码并按顺序拼接，去掉开头的 `flag=`，最后用 URL 的 `unquote_plus` 解码，得到原始 flag。

## 方法总结

字符黑名单没有解决上下文问题：双引号足以逃逸 JavaScript 字符串，而短载荷又能借助持久化的 `window.name` 引入任意长度脚本。同编号 Bot 与选手共享内存记录，使 `/result` 同时成为执行点和回传信道。完整利用需处理成绩初始化、25 字符分块、评论被覆盖后的重新植入，以及表单编码对 Base64 加号的转换。
