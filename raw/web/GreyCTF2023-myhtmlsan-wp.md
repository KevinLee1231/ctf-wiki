# GreyCTF 2023 MyHtmlSan

## 题目简述

服务允许托管用户提交的 HTML，并试图用正则表达式删除标签和注释。管理员机器人访问上报页面时设置可由 JavaScript 读取的 `session` Cookie，其中保存 flag。构造浏览器与正则表达式解析不同的畸形标签，可让一个带 `onerror` 的 `img` 元素在净化后存活。

## 解题过程

标签过滤使用一个正则从 `<` 匹配到它认为的闭合 `>`，但 HTML 解析器还会执行错误恢复。官方给出的结构为：

```html
<1aaa a=<img onerror="XSS_CODE" src="a" /'>
```

外层以数字开头，不是正常 HTML 标签；单引号和内部 `<img` 又使正则与浏览器对属性边界产生不同理解。正则删除了错误的片段，浏览器最终仍构造出 `img` 节点。将 `XSS_CODE` 换成把 `document.cookie` 发送到自有接收端的代码，上传后取得 `/uploads/<uuid>.html`，再通过 `/report` 让管理员访问。

无效 `src` 触发 `onerror`，脚本读取非 HttpOnly 的 `session` Cookie并回传：

```text
grey{r3geX_1s_N0t_4_htm1_cee664daa169f7cdb53f87ab810ccb15}
```

## 方法总结

HTML 不是正则语言，浏览器还包含复杂的容错与重新分词规则。自制“删标签”正则无法与真实解析器保持一致。应使用维护良好的 HTML sanitizer、限定允许的元素和属性，并用 CSP 与 HttpOnly Cookie 降低净化遗漏的影响。
