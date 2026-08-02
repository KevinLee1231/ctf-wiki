# hi

## 题目简述

页面用全屏 canvas 随机绘制大量 `hi`，并在其下方放置一张静态 SVG。视觉噪声只遮挡显示，没有保护资源；HTML 源码直接暴露 `/secret-b888c3f2.svg` 路径，而 SVG 本身是可读 XML。

## 解题过程

查看页面源代码即可找到：

```html
<img src="/secret-b888c3f2.svg" alt="bye">
```

直接请求资源并从 XML 的 `aria-label` 或文本路径描述中提取 flag：

```python
import re
import requests

svg = requests.get(
    "https://TARGET/secret-b888c3f2.svg", timeout=10
).text
print(re.search(r"tjctf\{[^}]+\}", svg).group(0))
```

结果为：

```text
tjctf{pretty_canvas_577f7045}
```

该 SVG 只是将同一段 flag 轮廓化为路径，并在可访问性标签中保留明文；正文已经完整转写，无需保留纯文字图片。

## 方法总结

- Canvas 遮挡只影响页面渲染，不影响静态资源 URL、网络响应或 DOM/HTML 源码。
- SVG 是 XML 文本，检查 `text`、`aria-label`、注释和路径相关元数据通常比截图 OCR 更直接。
- Web 题的低成本首检应包括页面源、开发者工具网络列表和公开静态目录。
