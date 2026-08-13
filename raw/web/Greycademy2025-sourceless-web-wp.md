# Sourceless Web

## 题目简述

页面按钮只显示假 flag，但 HTML、CSS 和 JavaScript 各藏有一段真实内容。前两段位于注释中，第三段由 JavaScript 对 Base64 数据逐字节异或解码；需要检查全部前端资源，而不是只看渲染后的 DOM 文本。

## 解题过程

HTML 的隐藏区域旁有第一段注释：

```html
<!-- flag1: grey{1_l0v3_ -->
```

CSS 中有第二段：

```css
/* flag2: 1n5pEC11Ng_ */
```

JavaScript 定义函数 `z`：先调用 `atob` 做 Base64 解码，再把每个字节与 `0xaf` 异或。按钮只有在计数器达到 `0xcafebabe` 时才调用：

```javascript
secretFlag.innerText = z("ycPOyJyVj92bweufwvDc2/rJ6dXS");
```

无需点击数十亿次，直接在控制台调用 `z`，或离线复现：

```python
import base64

ciphertext = base64.b64decode("ycPOyJyVj92bweufwvDc2/rJ6dXS")
decoded = bytes(value ^ 0xaf for value in ciphertext).decode()
print(decoded)
```

输出为：

```text
flag3: r4nD0m_stUfFz}
```

去掉标签 `flag3: `，按 `flag1 + flag2 + flag3` 拼接，得到 `grey{1_l0v3_1n5pEC11Ng_r4nD0m_stUfFz}`。本地直接解析三份原始资源并重算异或，结果与仓库 flag 一致。

## 方法总结

“无源码”只意味着服务端源码未提供，浏览器仍必须下载 HTML、CSS 和 JavaScript。检查注释、静态资源和事件处理器后，前端混淆通常可以被直接调用或离线等价实现。最终 flag 为 `grey{1_l0v3_1n5pEC11Ng_r4nD0m_stUfFz}`。
