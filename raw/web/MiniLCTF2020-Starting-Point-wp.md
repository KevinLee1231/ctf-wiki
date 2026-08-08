# MiniLCTF2020 - Starting Point

## 题目简述

这是一道 Web 入门签到题。页面可见区域没有给出有效线索，但 flag 已作为普通 HTML 文本写入页面，只是通过样式放在不显眼的位置。决定性步骤是检查浏览器实际收到的页面源码，而不是继续枚举接口。

## 解题过程

用“查看网页源代码”或开发者工具的 Elements 面板搜索 `minil{`。原题页面中可以直接看到类似下面的节点：

```html
<p style="color:#f9f9f9; position:fixed; left:5px; bottom:5px;">
  minil{Welcome_to_miniLCTF2020}
</p>
```

文本颜色接近背景色，因此直接浏览页面时容易忽略。读取节点内容即可得到：

```text
minil{Welcome_to_miniLCTF2020}
```

## 方法总结

Web 签到题先检查响应正文、HTML 注释、隐藏节点、前端脚本和静态资源，不要只看渲染结果。CSS 只能改变显示方式，不能从客户端收到的源码中抹去内容。
