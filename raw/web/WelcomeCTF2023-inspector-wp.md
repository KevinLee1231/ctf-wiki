# inspector

## 题目简述

题目是一张静态个人简历页面，Flag 被拆成三段，分别藏在 HTML 注释、JavaScript 注释和 CSS 注释中。页面显示内容本身没有漏洞，目标是检查浏览器加载的前端源文件。

## 解题过程

查看页面源代码，在 `index.html` 中找到：

```html
<!-- greyhats{1_4 -->
```

继续查看页面引用的脚本，`scripts/aos.js` 末尾包含：

```javascript
//m_4n_ins
```

最后在 `css/main.css` 找到：

```css
/* p3t0r_n0w} */
```

按照语义顺序连接三段：

```text
greyhats{1_4 + m_4n_ins + p3t0r_n0w}
```

得到：

```text
greyhats{1_4m_4n_insp3t0r_n0w}
```

## 方法总结

- 核心技巧：检查 HTML、CSS、JavaScript 三类静态资源中的注释。
- 识别信号：纯静态站点、题名提示 Inspector、README 明确把解法分为 HTML/CSS/JS 三部分。
- 复用要点：不要只看 DOM Elements 面板；还应检查原始响应、Sources 和页面实际加载的所有本地资源。
