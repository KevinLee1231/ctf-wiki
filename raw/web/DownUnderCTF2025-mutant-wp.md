# mutant

## 题目简述

页面把 `input` 查询参数先写入 `<template>`，遍历节点删除全部属性，并删除名称长度为 6 或 8 的元素，随后把 template 的 `innerHTML` 再写入真实 DOM。管理员机器人登录后把 flag 放入 `HttpOnly` 属性未设置的 `flag` cookie，并会访问通过 `/report` 提交的 payload。因此需要绕过的是自制 sanitizer，而不是寻找普通 `onerror` 是否会被直接保留。

```javascript
const t = document.createElement('template');
t.innerHTML = inp;
// 遍历并删除 attributes；节点名长度为 6 或 8 时删除节点
myoutput.innerHTML = t.innerHTML;
```

HTML 的解析结果并非在 template 与真实 DOM 两个上下文中完全不变。特制 MathML/HTML 嵌套会在第二次解析时发生 mutation：首次遍历时的节点与属性过滤看起来安全，序列化后重新赋给 `innerHTML` 却生成带事件处理器的 `<img>`。

## 解题过程

### 构造 mutation XSS

题目名和提示指向 DOMPurify 曾受影响的 MathML mutation XSS 族。常见链会用 `mglyph`，但此题针对标签名长度的过滤会删除它；使用同一解析特性的 `malignmark` 可保留必要结构：

```html
<form><math><mtext></form><form><malignmark><style></math><img src=1 onerror="PAYLOAD">
```

首次将该字符串放入 `template.innerHTML` 时，过滤循环删除它当前看到的属性和若干节点；浏览器对混合 `form`、MathML、`style` 的容错解析及序列化会改变树结构。最终赋值给 `myoutput.innerHTML` 时，`img` 的 `onerror` 才重新成为可执行事件属性。调试时应比对页面中已有的三次日志：原输入、template 序列化结果、最终 DOM，而不是只根据源字符串判断。

### 触发管理员并获取 cookie

把 `PAYLOAD` 替换为将 `document.cookie` 发往攻击者收集端的短脚本，然后把完整 HTML 作为 `/report` 的 POST body。服务端会构造管理员访问的 `/?input=<URL 编码 payload>`；机器人登录流程已经设置了 flag cookie，脚本可直接读取并带出它。

收集端和临时地址与题目无关，故不写入 WP；验证时只需确认报告后的管理员访问触发一次外带请求。

### 验证

成功外带的 cookie 值为：

```text
DUCTF{if_y0u_d1dnt_us3_mutation_x5S_th3n_it_w45_un1nt3nded_435743723}
```

## 方法总结

- 核心技巧：利用 HTML/MathML 的解析—序列化—再解析差异绕过只检查中间 DOM 的 sanitizer。
- 识别信号：应用先把输入塞进 `template` 或 detached DOM，再序列化并二次写入 `innerHTML`，且过滤规则依赖节点名称/当前树形时，应测试 mutation XSS。
- 复用要点：不要自行实现黑名单 sanitizer；应使用已修复的成熟库，并在最终插入位置做上下文正确的输出编码。对管理员机器人，敏感 cookie 必须使用 `HttpOnly`、`SameSite` 等纵深保护。
