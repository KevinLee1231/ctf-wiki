# 你好，爪洼脚本

## 题目简述

页面把一段颜文字形式的 JavaScript 放在 `<h2>` 标签中，标题提示将执行结果用 `0xGame{...}` 包裹。该表达式属于 AAEncode 风格混淆：利用 JavaScript 的隐式类型转换和特殊 Unicode 标识符拼出可执行代码。

## 解题过程

源码中的颜文字只是 `<h2>` 的文本内容，直接打开页面只会显示它，不会由浏览器自动执行。在仅处理本地题目源码的前提下，可在开发者工具控制台读取并执行该文本：

```javascript
eval(document.querySelector('h2').textContent)
```

这类 AAEncode 表达式先通过布尔值、数字和对象转字符串得到字符片段，再拼出 `constructor` 等属性名，取得 `Function` 构造器；后半段以转义数字序列还原真正的 JavaScript，最后调用它。

在隔离的 JavaScript 运行环境中拦截 `alert` 后，其参数精确为：

```text
Hello, JavaScript
```

弹窗内容中的标点是半角逗号，逗号后还有一个半角空格。按页面 `<title>` 给出的格式包裹，flag 为：

```text
0xGame{Hello, JavaScript}
```

## 方法总结

- 识别信号：大量颜文字标识符、隐式类型转换和动态 `Function` 调用通常指向 AAEncode。
- HTML 标签中的 JavaScript 字符串不会自动执行；需要区分“页面文本”和 `<script>` 中的代码。
- 对混淆代码应在隔离环境中拦截输出函数，准确保留空格、半角标点等不可忽略的字符。
