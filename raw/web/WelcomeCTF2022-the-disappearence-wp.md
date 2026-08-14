# the disappearence

## 题目简述

题目部署的是静态 Nginx 站点。页面视觉内容是 Lambda 公司的介绍，Flag 没有经过编码或脚本生成，而是直接写在首页“Latest News”列表项的 HTML 文本中。

## 解题过程

打开浏览器开发者工具的 Elements/Sources，或直接下载首页源码并搜索 Flag 前缀：

```bash
curl -s http://target/ | grep -n "greyhats"
```

在 `id="kidnapping"` 的新闻条目中可以看到：

```html
<li class="list-group-item" id="kidnapping">
  We have Inspector Gadget, he shall stop us no more!
  greyhats{w3_h4v3_th3_1n5p3ct0r}.
</li>
```

因此 Flag 为：

```text
greyhats{w3_h4v3_th3_1n5p3ct0r}
```

## 方法总结

静态 Web 题先检查实际响应和客户端源码，不要被页面布局、图片或故事背景带偏。浏览器渲染的所有 HTML、CSS、JavaScript 都会发送给客户端，隐藏元素、注释和不显眼的列表项并不构成保密措施；全文搜索 Flag 格式通常是成本最低的首轮检查。
