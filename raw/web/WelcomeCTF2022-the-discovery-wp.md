# the discovery

## 题目简述

题目是 Inspector Gadget 的静态个人主页。Flag 被拆成三段，分别藏在 HTML、站点自定义 CSS 和 JavaScript 的注释中。第三方压缩库和图片只是页面依赖，不需要逐个分析。

## 解题过程

从首页源码开始搜索 `Part` 或 `greyhats`，在 `index.html` 的推荐人区域找到：

```html
<!-- Part 1 / 3: greyhats{1_4m -->
```

继续检查首页实际引用的自定义资源，而不是体积较大的第三方库。在 `css/main.css` 与 `scripts/main.js` 中分别找到：

```css
/* Part 2 / 3: _th3_in5p */
```

```javascript
// Part 3 / 3: 3ct0r}
```

按注释标明的序号原样拼接，不增加或删除下划线：

```text
greyhats{1_4m_th3_in5p3ct0r}
```

也可以在下载后的站点目录执行：

```bash
grep -RIn --exclude='*.min.*' 'Part [123] / 3' .
```

排除压缩依赖能减少大量无关匹配。

## 方法总结

静态站点的秘密可能跨 HTML、CSS、JavaScript 分片。先从入口文档列出真正加载的自定义资源，再针对注释标记和 Flag 前缀搜索，比无差别检查所有依赖更高效。分片带有显式序号时应按序号拼接，并保留边界处的标点与下划线。
